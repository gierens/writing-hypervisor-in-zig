# シリアルコンソールによるログ出力

前回で Ymir が起動しました。
まず最初にやることは、何はともあれログ出力です。
[Surtr のとき](../bootloader/uefi_log.md)と同様ですね。

ログ出力にはシリアルポート[^serial]を使います。
`build.zig` で設定した起動オプションにより、QEMU はシリアルポートを標準出力にリダイレクトしてくれます。
このログ出力が実装できれば続く開発が楽になります。
なお、ログシステムの構築自体は次チャプターで行うこととして、本チャプターではシリアル出力すること自体を目標とします。

## Table of Contents

<!-- toc -->

## arch ディレクトリの作成

### x86 ディレクトリ

[Surtr のとき](../bootloader/simple_pg.md) と同様に、アーキテクチャに強く依存するコードはディレクトリを分けて実装します。
Ymir のディレクトリは以下のようになります:

```sh
tree ./ymir

./ymir
├── arch
│   └── x86
│       ├── arch.zig
│       ├── asm.zig
│       └── serial.zig
├── arch.zig
├── linker.ld
└── main.zig
```

`ymir/arch.zig` の内容は `surtr/arch.zig` と同じです。
`ymir/arch/x86/arch.zig` を上位のディレクトリに export する目的です。

`ymir/arch/x86/arch.zig` は、`ymir/arch/x86` ディレクトリにおけるルートです。
アーキテクチャ依存のコードを `ymir/arch` より上のディレクトリから利用する際には、必ずこのファイルを import して使うことにします。
`ymir/arch/x86/arch.zig` は以下のようにしておきます:

```ymir/arch/x86/arch.zig
pub const serial = @import("serial.zig");
const am = @import("asm.zig");
```

`serial.zig` は `pub` なので `arch.serial` として上位ディレクトリからアクセスできます。
`asm.zig` は `pub` がついていないため、`arch` ディレクトリ以外からアクセスできません。
アーキテクチャ依存のコードをできるだけ隠したいため、アセンブリ関連のコードは `asm.zig` にまとめて隠していきます。

### `ymir` モジュールの作成

現在のところ、`ymir/hoge/fuga.zig` から `ymir/piyo/neko.zig` を参照するためには、
以下のように相対パスで指定する必要があります:

```zig
const fuga = @import("../piyo/neko.zig");
```

これは見た目が気持ち悪いだけではなく、以下のように **誤って秘匿したいモジュールを参照してしまう可能性** があります:

```zig
const am = @import("arch/x86/asm.zig"); // 本来はアクセスさせたくない
```

これを防ぐため、ルートモジュールを作成し、**全てのモジュールはルートモジュールを経由してアクセスさせる** ことにします。
モジュールの作成は Surtr のときと同様に `build.zig` で定義します:

```build.zig
const ymir_module = b.createModule(.{
    .root_source_file = b.path("ymir/ymir.zig"),
});
ymir_module.addImport("ymir", ymir_module);
ymir_module.addImport("surtr", surtr_module);
```

定義した `ymir` モジュールを `ymir` 実行ファイルに追加します:

```build.zig
ymir.root_module.addImport("ymir", ymir_module);
```

これで、`@import("ymir")` のようにモジュール名で Ymir モジュールを import できるようになりました。
モジュールのルートである `ymir/ymir.zig` は、必要な全ての子モジュールを export します:

```ymir/ymir.zig
pub const arch = @import("arch.zig");
```

これで、全てのファイルから `@import("ymir").arch` のようにして `arch/x86/arch.zig` にアクセスできるようになりました。
試しに、`arch/x86/arch.zig` で適当な関数を定義して `main.zig` からアクセスしてみましょう:

```zig
// -- ymir/arch/x86/arch.zig --
pub fn someFunction() void {}

// -- ymir/main.zig --
const ymir = @import("ymir");
const arch = ymir.arch;
arch.someFunction();
```

`std` と同じノリで `ymir` モジュールにアクセスできますね。
これ以降は意図しない import を防ぐため、**同一ディレクトリ以外のファイルを相対パスを使って直接 import することは原則禁止**とします。

> [!TIP] ymir への ymir の追加
> Zig に馴染みがない人は、`ymir` executable に `ymir` モジュールを追加するというのは気持ち悪いかもしれません。
> 筆者も当初は違和感を感じたため [Ziggit](https://ziggit.dev/) で聞いてみたところ、このような書き方は合法なのは勿論、自然であるとのことでした[^self-dependent]。

## シリアルの基本の定義

今回想定するのは [8250 UART](https://en.wikipedia.org/wiki/8250_UART) です。
入力と出力のどちらもできますが、このチャプターでは出力のみを扱います。

シリアルポートにおける定数や構造体を定義していきます。
まずは COM portです。
COM port は I/O port を介してアクセスします。
COM port と I/O port の対応関係を定義するため、`ymir/arch/x86/serial.zig` を作成します:

```ymir/arch/x86/serial.zig
pub const Ports = enum(u16) {
    com1 = 0x3F8,
    com2 = 0x2F8,
    com3 = 0x3E8,
    com4 = 0x2E8,
};
```

今回は4つだけポートを定義します。
実際のHWによってはもっと多いかもしれませんし、もっと少ないかもしれません。
Ymir では COM1 を使います。

各ポートはそれぞれの data register を持っています。
Data register へは COM port をベースとしたオフセットでアクセスします:

```ymir/arch/x86/serial.zig
const offsets = struct {
    /// Transmitter Holding Buffer: DLAB=0, W
    pub const txr = 0;
    /// Receiver Buffer: DLAB=0, R
    pub const rxr = 0;
    /// Divisor Latch Low Byte: DLAB=1, R/W
    pub const dll = 0;
    /// Interrupt Enable Register: DLAB=0, R/W
    pub const ier = 1;
    /// Divisor Latch High Byte: DLAB=1, R/W
    pub const dlm = 1;
    /// Interrupt Identification Register: DLAB=X, R
    pub const iir = 2;
    /// FIFO Control Register: DLAB=X, W
    pub const fcr = 2;
    /// Line Control Register: DLAB=X, R/W
    pub const lcr = 3;
    /// Line Control Register: DLAB=0, R/W
    pub const mcr = 4;
    /// Line Status Register: DLAB=X, R
    pub const lsr = 5;
    /// Modem Status Register: DLAB=X, R
    pub const msr = 6;
    /// Scratch Register: DLAB=X, R/W
    pub const sr = 7;
};
```

実際には、オフセット・アクセスが read/write のどちらなのか・その時の **DLAB** の値 の3要素によってアクセスするレジスタが異なります。
どのような場合にどのレジスタにアクセスするのかは、`offsets` 内のコメントまたは参考文献[^serial]を参照してください。

## 初期化

シリアルの初期化を行います。

COM port へのアクセスは、対応する I/O port への `in`/`out` 命令を使っておこないます。
必要なアセンブリ命令を定義しましょう:

```ymir/arch/x86/asm.zig
pub inline fn inb(port: u16) u8 {
    return asm volatile (
        \\inb %[port], %[ret]
        : [ret] "={al}" (-> u8),
        : [port] "{dx}" (port),
    );
}

pub inline fn outb(value: u8, port: u16) void {
    asm volatile (
        \\outb %[value], %[port]
        :
        : [value] "{al}" (value),
          [port] "{dx}" (port),
    );
}
```

この関数を利用し、I/O port に読み書きしてシリアルを初期化します。
以下の表のようにレジスタを設定します[^am]。
なお、各レジスタは全て 8bit です:

| Register | Description | Value |
| --- | --- | --- |
| LC: Line Control | Line Protocol | **8n1** (8 data bit / No parity / 1 stop bits) |
| IE: Interrupt Enable | 有効化する割り込み | 0 (全ての割り込みを無効化) |
| FC: FIFO Control | FIFO バッファ | 0 (FIFO を無効化) |

```ymir/arch/x86/serial.zig
const am = @import("asm.zig");

pub fn initSerial(port: Ports, baud: u32) void {
    const p = @intFromEnum(port);
    am.outb(0b00_000_0_00, p + offsets.lcr); // 8n1: no paritiy, 1 stop bit, 8 data bit
    am.outb(0, p + offsets.ier); // Disable interrupts
    am.outb(0, p + offsets.fcr); // Disable FIFO
    ...
}
```

続いて、[Baud Rate](https://en.wikipedia.org/wiki/Baud) を設定します。
Baud Rate はその信号線に1秒間に送信されるビット数です。
各文字データの前には **Start Bit** が、後には **Stop Bit** (今回は`1`)が付加されます。
そのため 8n1 の場合、1文字(8bit)を送信するのに必要なビット数は 10 です。
送信したデータの 80% がデータとなります。
安定して送信できる最大の Baud Rate は `115200` らしいので、Ymir でもこの値を使います。

UART は `115200`/sec で動作するクロックを持っています。
UEFI はこのクロック周波数を、**Divisor** として設定された値で割ることで Baud Rate を計算します:

\\[ \text{Baud Rate} = \frac{115200} { \text{Divisor} } \\]

よって、Baud Rate として \\(\text{B}\\) を設定したい場合には、以下のように Divisor を計算します:

\\[ \text{Divisor} = \frac{115200} { \text{B} } \\]

以下のように Baud Rate を設定します:

```ymir/arch/x86/serial.zig
{
    ...
    const divisor = 115200 / baud;
    const c = am.inb(p + offsets.lcr);
    am.outb(c | 0b1000_0000, p + offsets.lcr); // Enable DLAB
    am.outb(@truncate(divisor & 0xFF), p + offsets.dll);
    am.outb(@truncate((divisor >> 8) & 0xFF), p + offsets.dlm);
    am.outb(c & 0b0111_1111, p + offsets.lcr); // Disable DLAB
}
```

LCR 中の *DLAB: Divisor Latch Access Bit* をセットすると、
*DLL: Divisor Latch Low* と *DLM: Divisor Latch High* にアクセスできるようになります。
DLAB をセットしたあとで、計算した `divisor` の下位・上位バイトをそれぞれ DLL と DLM に書き込みます。
`c` は DLAB を設定する前の LCR の値であり、divisor の設定後に LCR を復元するのに使っています。

## 文字の書き込み

続いて、初期化したシリアルになにか文字を書き込んでみます。

シリアルに書き込むためには、*TX-buffer* が空になるのを待つ必要があります。
TX-buffer が空なのかどうかは、 *LSR: Line Status Register* の *THRE: Transmitter Holding Register Empty* ビットで確認できます。
もしも空でなかった場合には、空になるまで待ちます:

```ymir/arch/x86/serial.zig
const bits = ymir.bits;

pub fn writeByte(byte: u8, port: Ports) void {
    // Wait until the transmitter holding buffer is empty
    while (!bits.isset(am.inb(@intFromEnum(port) + offsets.lsr), 5)) {
        am.relax();
    }

    // Put char to the transmitter holding buffer
    am.outb(byte, @intFromEnum(port));
}
```

`bits` はビット計算のためのライブラリです。
[ビット演算とテスト](bit_and_test.md) でまとめて定義します。
今は `isset(B, N)` で `B` の `N` ビット目がセットされているかどうかを確認できると考えてください。
LSR の 5-th bit が THRE であるため、これがセットされるまで待ちます。
`am.relax()` は `rep; nop` をするアセンブリ関数です。
少しだけCPUを休ませるために使います。

TX-buffer が空になったら、引数で渡された `byte` を COM port に書き込みます。
オフセット `0` が TX-buffer に対応しているため、`am.outb(byte, @intFromEnum(port))` となります。

実際に書き込めるかどうか確認してみましょう。
`main.zig` に実権コードを追加します:

```zig
const ymir = @import("ymir");
const arch = ymir.arch;

arch.serial.initSerial(.com1, 115200);
for ("Hello, Ymir!\n") |c|
    arch.serial.writeByte(c, .com1);
```

実行してみて、`Hello, Ymir!` が表示されれば成功です。

## `Serial` ラッパークラス

シリアル出力はできるようになったものの、`Hello, Ymir!` という文字列を書き込むためにわざわざ `for` ループするのはめんどうです。
また、`arch` 以下のファイルを直接呼び出すのも少しアーキ依存が強すぎる気がします。
というわけで、生のシリアルをラップする `Serial` クラスを作成します[^class]。

`arch` を抽象化するため、ルート直下に `ymir/serial.zig` を作成します:

```ymir/serial.zig
const ymir = @import("ymir");
const arch = ymir.arch;

pub const Serial = struct {
    const Self = @This();
    const WriteFn = *const fn (u8) void;
    const ReadFn = *const fn () ?u8;

    _write_fn: WriteFn = undefined,
    _read_fn: ReadFn = undefined,
    ...
};
```

`Serial` は、シリアル出力・入力用にそれぞれ `_write_fn` と `_read_fn` という関数ポインタを持ちます[^input]。
`Serial` は以下のように Baud Rate は `115200` としてインスタンス化します:

```ymir/serial.zig
pub fn init() Serial {
    var serial = Serial{};
    arch.serial.initSerial(&serial, .com1, 115200);
    return serial;
}
```

空の `Serial` 構造体を作ったあと、`initSerial()` に渡しています。
先ほど実装した `initSerial()` を第1引数として `*Serial` を受け取れるように変更します:

```ymir/arch/x86/serial.zig
pub fn initSerial(serial: *Serial, port: Ports, baud: u32) void {
    ...
    serial._write_fn = switch (port) {
        .com1 => writeByteCom1,
        .com2 => writeByteCom2,
        .com3 => writeByteCom3,
        .com4 => writeByteCom4,
    };
}
```

`_write_fn` にアーキ依存のシリアル出力関数を設定しています。
`writeByteComN()` は `Port` に対応する COM port に出力するためのヘルパー関数で、実体は `writeByte()` です:

```ymir/arch/x86/serial.zig
fn writeByteCom1(byte: u8) void {
    writeByte(byte, .com1);
}
...
```

これで `Serial` に出力用の関数を設定できました。
利用者側からは使いやすいように、1文字を出力するための関数と文字列を出力する関数を提供しましょう:

```ymir/serial.zig
pub fn write(self: Self, c: u8) void {
    self._write_fn(c);
}

pub fn writeString(self: Self, s: []const u8) void {
    for (s) |c| {
        self.write(c);
    }
}
```

これで `Serial` は完成です。
`main.zig` において、以下のように初期化して使ってみます:

```ymir/main.zig
const sr = serial.init();
sr.writeString("Hello, Ymir!\n");
```

先ほどと同様に `Hello, Ymir!` が表示されれば成功です。
本チャプターでは、シリアル出力をするための基本的な機能を実装しました。
これをもとにすると、Surtr で実装したのと同様なログシステムを作ることができます。
次チャプターではログシステムの実装します。

[^serial]: [Serial Ports - OSDev Wiki](https://wiki.osdev.org/Serial_Ports)
[^self-dependent]: [ZLS does not work for @import(“root”) when multiple artifacts are installed - Ziggit](https://ziggit.dev/t/zls-does-not-work-for-import-root-when-multiple-artifacts-are-installed/4190)
[^am]: コード中で `asm.zig` を `am` として import しているのは、`asm` が Zig の[予約語](https://ziglang.org/documentation/master/#Keyword-Reference)であるためです。
[^class]: Zig にはクラスというものは無く、構造体しかありません。
しかし、クラスといった方が伝わりやすい気がするため、ここではクラスと呼びます。
[^input]: なお、シリアル入力はずっと先のチャプターで扱います。