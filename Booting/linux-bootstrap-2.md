# Kernel booting Prozess. Part 2.

## Erste Schritte beim Kernel-Setup

Wir hatten vorherigen [Teil](linux-bootstrap-1.md) damit angefangen, in die Internas des Linux-Kernels einzutauchen und sahen den ersten Teil des Kernel-Setup-Codes. Wir sind bis zum ersten Aufruf der `main`-Funktion (die erste in C geschriebene Funktion) [arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c) gekommen.

In diesem Teil werden wir den Kernel-Setup-Code weiter untersuchen und folgende Dinge durchgehen

- Was ist der `protected mode`?
- der Übergang dorhin,
- die Initialisierung des Heaps und der Konsole,
- Speichererkennung, CPU-Validierung und Tastaturinitialisierung
- und viel viel mehr.

Also, lass uns weitermachen.

## Sicherheitsmodus

Bevor wir in den nativen Intel64 [Long Mode](http://en.wikipedia.org/wiki/Long_mode) wechseln können, muss der Kernel die CPU in den geschützten Modus versetzen.

Was ist der [geschützte Modus](https://en.wikipedia.org/wiki/Protected_mode)? Der geschützte Modus wurde erstmals 1982 zur x86-Architektur hinzugefügt und war der Hauptmodus von Intel-Prozessoren vom Prozessor [80286](http://en.wikipedia.org/wiki/Intel_80286) bis Intel 64 und dem Long Mode.

Der Hauptgrund für die Abkehr vom [Real-Modus](http://wiki.osdev.org/Real_Mode) ist der sehr eingeschränkte Zugriff auf den Arbeitsspeicher. Wie Sie sich vielleicht aus dem vorherigen Teil erinnern, stehen im Real-Modus nur 2<sup>20</sup> Bytes oder 1 Megabyte, manchmal sogar nur 640 Kilobyte RAM zur Verfügung.

Der geschützte Modus brachte viele Änderungen mit sich, aber der wichtigste ist der Unterschied in der Speicherverwaltung. Der 20-Bit-Adressbus wurde durch einen 32-Bit-Adressbus ersetzt. Es ermöglichte den Zugriff auf 4 Gigabyte Speicher im Vergleich zu 1 Megabyte im Real-Modus. Außerdem wurde die Unterstützung für [Paging](http://en.wikipedia.org/wiki/Paging) hinzugefügt, über die Sie in den nächsten Abschnitten mehr erfahren können.

Die Speicherverwaltung im geschützten Modus ist in zwei nahezu unabhängige Teile unterteilt:

- Segmentierung
- Paging

Hier werden wir nur über Segmentierung sprechen. Paging wird in den nächsten Abschnitten behandelt.

Wie Sie im vorherigen Teil lesen können, bestehen Adressen im Real-Modus aus zwei Teilen:

- Basisadresse des Segments
- Versatz von der Segmentbasis

Wir können die physikalische Adresse schließen, wenn wir diese beiden Dinge kennen:

```
PhysicalAddress = Segment Selector * 16 + Offset
```

Die Speichersegmentierung wurde im geschützten Modus komplett überarbeitet. Es gibt keine 64-Kilobyte-Segmente mit fester Größe. Stattdessen wird die Größe und Position jedes Segments durch eine zugeordnete Datenstruktur beschrieben, die als _Segment Descriptor_ bezeichnet wird. Die Segmentdeskriptoren werden in einer Datenstruktur gespeichert, die als `Global Descriptor Table` (GDT) bezeichnet wird.

Die GDT ist eine Struktur, die sich im Speicher befindet. Sie hat keinen festen Platz im Speicher, daher wird seine Adresse im speziellen `GDTR`-Register gespeichert. Später werden wir sehen, wie der GDT in den Linux-Kernel-Code geladen wird. Es wird eine Operation zum Laden in den Speicher geben, etwa wie:

```assembly
lgdt gdt
```

wobei der Befehl "lgdt" die Basisadresse und die Grenze (Größe) der globalen Deskriptortabelle in das Register `GDTR` lädt. `GDTR` ist ein 48-Bit-Register und besteht aus zwei Teilen:

- die Größe (16 Bit) der globalen Deskriptortabelle;
- die Adresse (32 Bit) der globalen Deskriptortabelle.

Wie oben erwähnt, enthält die GDT `segment descriptors`, welche Speichersegmente beschreiben. Jeder Deskriptor hat eine Größe von 64 Bit. Das allgemeine Schema eines Deskriptors lautet:

```
 63         56         51   48    45           39        32
------------------------------------------------------------
|             | |B| |A|       | |   | |0|E|W|A|            |
| BASE 31:24  |G|/|L|V| LIMIT |P|DPL|S|  TYPE | BASE 23:16 |
|             | |D| |L| 19:16 | |   | |1|C|R|A|            |
------------------------------------------------------------

 31                         16 15                         0
------------------------------------------------------------
|                             |                            |
|        BASE 15:0            |       LIMIT 15:0           |
|                             |                            |
------------------------------------------------------------
```

Keine Sorge, ich weiß, dass es nach dem Real-Modus ein wenig beängstigend aussieht, aber es ist einfach. Zum Beispiel bedeutet LIMIT 15: 0, dass sich die Bits 0-15 von Limit sich am Anfang des Deskriptors befinden. Der Rest befindet sich an LIMIT 19:16, das sich in den Bits 48-51 des Deskriptors befindet. Die Größe von Limit ist also 0-19, d. H. 20 Bit. Schauen wir es uns genauer an:

1. Das Limit [20-Bit] wird zwischen den Bits 0-15 und 48-51 aufgeteilt. Es definiert die `length_of_segment - 1`. Dies hängt vom `G`-Bit (Granularitäts-Bit) ab.

- Wenn `G` (Bit 55) 0 ist und die Segmentgrenze 0 ist, beträgt die Größe des Segments 1 Byte
- Wenn `G` 1 ist und das Segmentlimit 0 ist, beträgt die Größe des Segments 4096 Bytes
- Wenn `G` 0 ist und das Segmentlimit 0xfffff ist, beträgt die Segmentgröße 1 Megabyte
- Wenn `G` 1 und das Segmentlimit 0xfffff ist, beträgt die Segmentgröße 4 Gigabyte

Das heißt also:

- Wenn G 0 ist, wird Limit in Form von 1 Byte interpretiert und die maximale Größe des Segments kann 1 Megabyte betragen.
- Wenn G 1 ist, wird Limit in Form von 4096 Bytes = 4 KBytes = 1 Page interpretiert und die maximale Größe des Segments kann 4 Gigabytes betragen. Tatsächlich wird, wenn G 1 ist, der Wert von Limit um 12 Bits nach links verschoben. Also 20 Bits + 12 Bits = 32 Bits und 2<sup>32</sup> = 4 Gigabyte.

2. Die Basis [32 Bits] wird auf die Bits 16-31, 32-39 und 56-63 aufgeteilt. Es definiert die physikalische Adresse des Startorts des Segments.

3. Typ/Attribut[5 Bits] wird durch die Bits 40-44 dargestellt. Es definiert die Art des Segments und wie darauf zugegriffen werden kann.

- Das `S`-Flag bei Bit 44 gibt den Deskriptortyp an. Wenn `S` 0 ist, ist dieses Segment ein Systemsegment, während `S` 1 ist, ist dies ein Code- oder Datensegment (Stapelsegmente sind Datensegmente, die Lese-/Schreibsegmente sein müssen).

Um festzustellen, ob es sich bei dem Segment um ein Code- oder Datensegment handelt, können Sie dessen Ex-Attribut (Bit 43) überprüfen (im obigen Diagramm mit 0 gekennzeichnet). Wenn es 0 ist, ist das Segment ein Datensegment, andernfalls ist es ein Codesegment.

Ein Segment kann von einem der folgenden Typen sein:

```
--------------------------------------------------------------------------------------
|           Type Field        | Descriptor Type | Description                        |
|-----------------------------|-----------------|------------------------------------|
| Decimal                     |                 |                                    |
|             0    E    W   A |                 |                                    |
| 0           0    0    0   0 | Data            | Read-Only                          |
| 1           0    0    0   1 | Data            | Read-Only, accessed                |
| 2           0    0    1   0 | Data            | Read/Write                         |
| 3           0    0    1   1 | Data            | Read/Write, accessed               |
| 4           0    1    0   0 | Data            | Read-Only, expand-down             |
| 5           0    1    0   1 | Data            | Read-Only, expand-down, accessed   |
| 6           0    1    1   0 | Data            | Read/Write, expand-down            |
| 7           0    1    1   1 | Data            | Read/Write, expand-down, accessed  |
|                  C    R   A |                 |                                    |
| 8           1    0    0   0 | Code            | Execute-Only                       |
| 9           1    0    0   1 | Code            | Execute-Only, accessed             |
| 10          1    0    1   0 | Code            | Execute/Read                       |
| 11          1    0    1   1 | Code            | Execute/Read, accessed             |
| 12          1    1    0   0 | Code            | Execute-Only, conforming           |
| 14          1    1    0   1 | Code            | Execute-Only, conforming, accessed |
| 13          1    1    1   0 | Code            | Execute/Read, conforming           |
| 15          1    1    1   1 | Code            | Execute/Read, conforming, accessed |
--------------------------------------------------------------------------------------
```

Wie wir sehen können, ist das erste Bit (Bit 43) `0` für ein _data_ Segment und `1` für ein _code_ Segment. Die nächsten drei Bits (40, 41, 42) sind entweder `EWA`(*E*xpansion *W*ritable *A*ccessible) oder CRA(*C*onforming *R*eadable *A*ccessible).

- Wenn E (Bit 42) 0 ist, erweitern Sie nach oben, andernfalls erweitern Sie nach unten. Lesen Sie [hier](http://www.sudleyplace.com/dpmione/expanddown.html) mehr dazu.
- Wenn W (Bit 41) (für Datensegmente) 1 ist, ist der Schreibzugriff zulässig, und wenn es 0 ist, ist das Segment schreibgeschützt. Beachten Sie, dass Lesezugriffe auf Datensegmente immer zulässig sind.
- A(Bit 40) steuert, ob der Prozessor auf das Segment zugreifen kann oder nicht.
- C(Bit 43) ist das übereinstimmende Bit (für Code-Selektoren). Wenn C 1 ist, kann der Segmentcode von einer niedrigeren Berechtigungsstufe (z. B. Benutzer) ausgeführt werden. Wenn C 0 ist, kann es nur mit derselben Berechtigungsstufe ausgeführt werden.
- R(Bit 41) steuert den Lesezugriff auf Codesegmente; Wenn es 1 ist, kann das Segment gelesen werden. Schreibzugriff wird niemals für Codesegmente gewährt.

4. DPL[2 Bits] (Descriptor Privilege Level) umfasst die Bits 45-46. Es definiert die Berechtigungsstufe des Segments. Es kann 0-3 sein, wobei 0 die am meisten privilegierte Stufe ist.

5. Das P-Flag (Bit 47) zeigt an, ob das Segment im Speicher vorhanden ist oder nicht. Wenn P 0 ist, wird das Segment als ungültig dargestellt, und der Prozessor lehnt es ab, aus diesem Segment zu lesen.

6. AVL-Flag (Bit 52) ​​- Verfügbare und reservierte Bits. Es wird unter Linux ignoriert.

7. Das L-Flag (Bit 53) zeigt an, ob ein Codesegment nativen 64-Bit-Code enthält. Wenn es gesetzt ist, wird das Codesegment im 64-Bit-Modus ausgeführt.

8. Das D/B-Flag (Bit 54) (Standard / Big-Flag) repräsentiert die Operandengröße, d. H. 16/32 Bits. Wenn es gesetzt ist, beträgt die Operandengröße 32 Bit. Ansonsten sind es 16 Bit.

Segmentregister enthalten Segmentselektoren wie im Real-Modus. Im geschützten Modus wird eine Segmentauswahl jedoch anders behandelt. Jedem Segmentdeskriptor ist ein Segmentselektor zugeordnet, der eine 16-Bit-Struktur aufweist:

```
 15             3 2  1     0
-----------------------------
|      Index     | TI | RPL |
-----------------------------
```

Wobei,

- **Index** speichert die Indexnummer des Deskriptors in der GDT.
- **TI** (Tabellenindikator) gibt an, wo nach dem Deskriptor gesucht werden soll. Wenn es 0 ist, wird der Deskriptor in der globalen Deskriptortabelle (GDT) gesucht. Andernfalls wird in der Local Descriptor Table (LDT) gesucht.
- Und **RPL** enthält die Berechtigungsstufe des Anforderers.

Jedes Segmentregister hat einen sichtbaren und einen versteckten Teil.

- Visible - Hier wird der Segmentselektor gespeichert.
- Hidden - Der Segmentdeskriptor (der die Basis, das Limit, die Attribute und die Flags enthält) wird hier gespeichert.

Die folgenden Schritte sind erforderlich, um eine physische Adresse im geschützten Modus abzurufen:

- Der Segmentselektor muss in einer der Segmentregister geladen sein.
- Die CPU versucht, einen Segmentdeskriptor am Versatz `GDT address + Index` aus dem Selektor zu finden und lädt dann den Deskriptor in den _hidden_ Teil des Segmentregisters.
- Wenn Paging deaktiviert ist, wird die lineare Adresse des Segments oder seine physikalische Adresse durch die folgende Formel angegeben: Basisadresse (im Deskriptor des vorherigen Schritts angegeben) + Versatz.

Schematisch wird es so aussehen:

![lineare Adresse](http://oi62.tinypic.com/2yo369v.jpg)

Der Algorithmus für den Übergang vom Realmodus in den geschützten Modus sieht wie folgt aus:

- Interrupts deaktivieren
- Beschreiben und laden des GDT mit der Anweisung `lgdt`
- Setzen des PE-Bits (Protection Enable) in CR0 (Control Register 0)
- Zum Code für den geschützten Modus springen

Wir werden den vollständigen Übergang zum geschützten Modus im Linux-Kernel im nächsten Teil sehen, aber bevor wir in den geschützten Modus wechseln können, müssen wir noch einige Vorbereitungen treffen.

Schauen wir uns [arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c) an. Wir können dort einige Routinen sehen, die eine Tastaturinitialisierung, eine Heap-Initialisierung usw. durchführen. Lassen Sie uns einen Blick darauf werfen.

## Boot-Parameter in die "zeropage" kopieren

Wir werden von der `main`-Routine in "main.c" ausgehen. Die erste Funktion, die in `main` aufgerufen wird, ist [`copy_boot_params (void)`](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c). Es kopiert den Kernel-Setup-Header in das entsprechende Feld der `boot_params`-Struktur, welche in der Header-Datei [arch/x86/include/uapi/asm/bootparam.h](https://github.com/torvalds/linux/blob/v4.16/arch/x86/include/uapi/asm/bootparam.h) definiert ist.

Die `boot_params` Struktur enthält das`struct setup_header hdr` Feld. Diese Struktur enthält dieselben Felder wie im [Linux-Boot-Protokoll](https://www.kernel.org/doc/Documentation/x86/boot.txt) definiert und wird vom Bootloader und dem Kernel zur compile/build Zeit ausgeführt. `copy_boot_params` macht zwei Dinge:

1. Es kopiert `hdr` von [header.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L280) in das Feld`setup_header` in der `boot_params` Struktur.

2. Der Zeiger auf die Kernel-Befehlszeile wird aktualisiert, wenn der Kernel mit dem alten Befehlszeilenprotokoll geladen wurde.

Beachten Sie, dass `hdr` mit der Funktion `memcpy` kopiert wird, die in der Quelldatei [copy.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/copy.S) definiert ist. Werfen wir einen Blick hinein:

```assembly
GLOBAL(memcpy)
    pushw   %si
    pushw   %di
    movw    %ax, %di
    movw    %dx, %si
    pushw   %cx
    shrw    $2, %cx
    rep; movsl
    popw    %cx
    andw    $3, %cx
    rep; movsb
    popw    %di
    popw    %si
    retl
ENDPROC(memcpy)
```

Ja, wir sind gerade zu C-Code und jetzt wieder zu Assembly übergegangen. Zunächst sehen wir, dass `memcpy` und andere hier definierte Routinen mit den beiden Makros `GLOBAL` und `ENDPROC` beginnen und enden. `GLOBAL` ist in [arch/x86/include/asm/linkage.h](https://github.com/torvalds/linux/blob/v4.16/arch/x86/include/asm/linkage.h) beschrieben, dies definiert die `globl` Direktive und dessen Label. `ENDPROC` wird in [include/linux/linkage.h](https://github.com/torvalds/linux/blob/v4.16/include/linux/linkage.h) beschrieben und kennzeichnet das `name`-Symbol als einen Funktionsnamen und endet mit der Größe des `name`-Symbols.

Die Implementierung von `memcpy` ist einfach. Zuerst werden die Werte aus den Registern `si` und `di` in den Stapel geschrieben, um ihre Werte beizubehalten, da sie sich während des `memcpy` ändern. Wie wir in `REALMODE_CFLAGS` in`arch/x86/Makefile` sehen können, verwendet das Kernel-Build-System die Option `-mregparm=3` von GCC, die Funktionen erhält seine ersten drei Parameter aus den `ax`, `dx` und `cx` Registern. Der Aufruf von `memcpy` sieht folgendermaßen aus:

```c
memcpy(&boot_params.hdr, &hdr, sizeof hdr);
```

Also,

- `ax` enthält die Adresse von `boot_params.hdr`
- `dx` enthält die Adresse von `hdr`
- `cx` enthält die Größe von `hdr` in bytes.

`memcpy` setzt die Adresse von `boot_params.hdr` nach `di` und speichert`cx` auf dem Stack. Danach wird der Wert zweimal nach rechts verschoben (oder durch 4 geteilt) und vier Bytes von der Adresse bei `si` an die Adresse bei `di` kopiert. Danach stellen wir die Größe von `hdr` wieder her, richten sie um 4 Bytes aus und kopieren den Rest der Bytes von der Adresse bei `si` zu der Adresse bei `di` byteweise (falls mehr vorhanden ist). Jetzt werden die Werte von `si` und `di` vom Stapel wiederhergestellt und der Kopiervorgang ist beendet.

## Konsoleninitialisierung

Nachdem `hdr` nach `boot_params.hdr` kopiert wurde, muss die Konsole im nächsten Schritt durch Aufrufen der Funktion `console_init` initialisiert werden, welche in [arch/x86/boot/early_serial_console.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/early_serial_console.c) definiert ist.

Es wird versucht, die Option `earlyprintk` in der Befehlszeile zu finden. War die Suche erfolgreich, werden die Portadresse und die Baudrate des seriellen Ports analysiert und der serielle Port initialisiert. Der Wert der Befehlszeilenoption `earlyprintk` kann einer der folgenden Werte sein:

- serial,0x3f8,115200
- serial,ttyS0,115200
- ttyS0,115200

Nach der Initialisierung der seriellen Schnittstelle sehen wir als erste Ausgabe:

```C
if (cmdline_find_option_bool("debug"))
    puts("early console in setup code\n");
```

Die Definition von `puts` befindet sich in [tty.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/tty.c). Wie wir sehen können, printet es Zeichen für Zeichen in einer Schleife, indem es die Funktion `putchar` aufruft. Schauen wir uns die Implementierung von `putchar` an:

```C
void __attribute__((section(".inittext"))) putchar(int ch)
{
    if (ch == '\n')
        putchar('\r');

    bios_putchar(ch);

    if (early_serial_base != 0)
        serial_putchar(ch);
}
```

`__attribute__((section(".inittext")))` bedeutet, dass sich dieser Code im Abschnitt `.inittext` befindet. Wir finden es in der Linker-Datei [setup.ld](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld).

Zuerst sucht `putchar` nach dem Symbol `\n` und gibt `\r` aus wenn es gefunden wird. Anschließend wird das Zeichen auf dem VGA-Bildschirm gedruckt, indem das BIOS mit dem Interrupt-Aufruf `0x10` aufgerufen wird:

```C
static void __attribute__((section(".inittext"))) bios_putchar(int ch)
{
    struct biosregs ireg;

    initregs(&ireg);
    ireg.bx = 0x0007;
    ireg.cx = 0x0001;
    ireg.ah = 0x0e;
    ireg.al = ch;
    intcall(0x10, &ireg, NULL);
}
```

Hier nimmt `initregs` die `Biosregs`-Struktur und füllt `Biosregs` zunächst mit der `memset`-Funktion mit Nullen und füllt sie dann mit Registerwerten.

```C
    memset(reg, 0, sizeof *reg);
    reg->eflags |= X86_EFLAGS_CF;
    reg->ds = ds();
    reg->es = ds();
    reg->fs = fs();
    reg->gs = gs();
```

Schauen wir uns die Implementierung von [memset](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/copy.S#L36) an:

```assembly
GLOBAL(memset)
    pushw   %di
    movw    %ax, %di
    movzbl  %dl, %eax
    imull   $0x01010101,%eax
    pushw   %cx
    shrw    $2, %cx
    rep; stosl
    popw    %cx
    andw    $3, %cx
    rep; stosb
    popw    %di
    retl
ENDPROC(memset)
```

Wie Sie oben lesen können, werden die gleichen Aufrufkonventionen wie bei der Funktion `memcpy` verwendet. Dies bedeutet, dass die Funktion ihre Parameter aus den Registern `ax`, `dx` und `cx` bezieht.

Die Implementierung von `memset` ähnelt der von memcpy. Es speichert den Wert des `di`-Registers auf dem Stapel und setzt den Wert von `ax`, der die Adresse der `biosregs`-Struktur nach `di` speichert. Als nächstes folgt der Befehl `movzbl`, der den Wert von `dl` in das unterste Byte des `eax`-Registers kopiert. Die restlichen 3 hohen Bytes von `eax` werden mit Nullen gefüllt.

Der nächste Befehl multipliziert `eax` mit `0x01010101`. Dies ist erforderlich, da `memset` 4 Bytes gleichzeitig kopiert. Wenn wir zum Beispiel eine Struktur mit einer Größe von 4 Bytes mit dem Wert `0x7` mit memset füllen müssen, enthält `eax` den Wert `0x00000007`. Wenn wir also `eax` mit `0x01010101` multiplizieren, erhalten wir `0x07070707`, und jetzt können wir diese 4 Bytes in die Struktur kopieren. `memset` benutzt die `rep; stosl` Anweisung um `eax` nach `es:di` zu kopieren.

Der Rest der `memset`-Funktion macht fast das Gleiche wie `memcpy`.

Nachdem die `biosregs`-Struktur mit `memset` gefüllt ist, ruft `bios_putchar` den Interrupt [0x10](http://www.ctyme.com/intr/rb-0106.htm) auf, der ein Zeichen ausgibt. Anschließend prüft es, ob die serielle Schnittstelle initialisiert wurde oder nicht und schreibt dort ein Zeichen mit [serial_putchar](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/tty.c) und `inb/outb` Anweisungen, wenn es gesetzt wurde.

## Heap-Initialisierung

Nachdem der Stack- und der BSS-Abschnitt in [header.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S) vorbereitet wurden (siehe vorheriger [Teil]) (linux-bootstrap-1.md)), muss der Kernel den [Heap](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c) über die Funktion [`init_heap`](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c) initialisieren.

Zunächst überprüft `init_heap` das Flag [`CAN_USE_HEAP`](https://github.com/torvalds/linux/blob/v4.16/arch/x86/include/uapi/asm/bootparam.h#L24) aus der [`loadflags`](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L320) Struktur im Kernel-Setup-Header und berechnet das Ende des Stacks sofern dieses Flag gesetzt war:

```C
    char *stack_end;

    if (boot_params.hdr.loadflags & CAN_USE_HEAP) {
        asm("leal %P1(%%esp),%0"
            : "=r" (stack_end) : "i" (-STACK_SIZE));
```

oder mit anderen Worten `stack_end = esp - STACK_SIZE`.

Dann gibt es die `heap_end` Berechnung:

```C
     heap_end = (char *)((size_t)boot_params.hdr.heap_end_ptr + 0x200);
```

was `heap_end_ptr` oder auch `_end` + `512` (`0x200h`) bedeutet. Die letzte Überprüfung ist, ob `heap_end` größer als`stack_end` ist. Wenn dies der Fall ist, wird `stack_end` `heap_end` zugewiesen, um sie gleich zu machen.

Jetzt ist der Heap initialisiert und wir können ihn mit der `GET_HEAP`-Methode verwenden. Wir werden in den nächsten Beiträgen sehen, wofür er verwendet wird, wie es verwendet wird und wie es implementiert wird.

## CPU-Validierung

Wie wir sehen können wäre der nächste Schritt die CPU-Validierung mit der Funktion `validate_cpu` aus der Quellcode-Datei [arch/x86/boot/cpu.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/cpucheck.c).

Es ruft die Funktion [`check_cpu`](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/cpucheck.c) auf und übergibt ihr die CPU-Stufe und die erforderliche CPU-Stufe und überprüft ob der Kernel auf der richtigen CPU-Ebene startet.

```C
check_cpu(&cpu_level, &req_level, &err_flags);
if (cpu_level < req_level) {
    ...
    return -1;
}
```

Die Funktion `check_cpu` prüft die Flags der CPU, das Vorhandensein vom [long mode](http://en.wikipedia.org/wiki/Long_mode) bei der x86_64(64-bit)-CPU, prüft den Hersteller des Prozessors und macht Vorbereitungen für bestimmte Anbieter, z. B. Deaktivieren von SSE+SSE2 für AMD, falls diese fehlen sollten usw.

Im nächsten Schritt wird möglicherweise ein Aufruf der Funktion `set_bios_mode` angezeigt, nachdem der Setup-Code herausgefunden hat, ob eine CPU dafür geeignet ist. Wie wir sehen, ist diese Funktion nur für den `x86_64`-Modus implementiert:

```C
static void set_bios_mode(void)
{
#ifdef CONFIG_X86_64
	struct biosregs ireg;

	initregs(&ireg);
	ireg.ax = 0xec00;
	ireg.bx = 2;
	intcall(0x15, &ireg, NULL);
#endif
}
```

Die Funktion `set_bios_mode` führt den BIOS-Interrupt`0x15` aus, um dem BIOS mitzuteilen, dass der [long mode](https://en.wikipedia.org/wiki/Long_mode) (falls `bx == 2`) verwendet wird.

## Speichererkennung

Der nächste Schritt ist die Speichererkennung mit der Funktion `detect_memory`. `detect_memory` stellt der CPU im Grunde eine map des verfügbaren RAM zur Verfügung. Es werden verschiedene Programmierschnittstellen zur Speichererkennung verwendet, z. B. `0xe820`, `0xe801` und `0x88`. Wir werden hier nur die Implementierung der **0xE820**-Schnittstelle sehen.

Sehen wir uns die Implementierung der Funktion `detect_memory_e820` in der Quelldatei [arch/x86/boot/memory.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/memory.c) an. Zunächst initialisiert die Funktion `detect_memory_e820` die oben gezeigte`Biosregs`-Struktur und füllt die Register mit speziellen Werten für den Aufruf `0xe820`:

```assembly
    initregs(&ireg);
    ireg.ax  = 0xe820;
    ireg.cx  = sizeof buf;
    ireg.edx = SMAP;
    ireg.di  = (size_t)&buf;
```

- `ax` enthält die Nummer der Funktion (in unserem Fall 0xe820)
- `cx` enthält die Größe des Puffers, der Daten über den Speicher enthält
- `edx` muss die magische Zahl `SMAP` enthalten
- `es:di` muss die Adresse des Puffers enthalten, der die Speicherdaten enthält
- `ebx` muss Null sein.

Als nächstes folgt eine Schleife, in der Daten über den Speicher gesammelt werden. Es beginnt mit einem Aufruf des BIOS-Interrupts `0x15`, der eine Zeile aus der Adresszuordnungstabelle schreibt. Um die nächste Zeile zu erhalten, müssen wir diesen Interrupt erneut aufrufen (was wir in der Schleife tun). Vor dem nächsten Aufruf muss `ebx` den zuvor zurückgegebenen Wert enthalten:

```C
    intcall(0x15, &ireg, &oreg);
    ireg.ebx = oreg.ebx;
```

Letztendlich sammelt diese Funktion Daten aus der Adresszuweisungstabelle und schreibt diese Daten in das Array `e820_entry`:

- Beginn des Speichersegments
- Größe des Speichersegments
- Art des Speichersegments (ob das bestimmte Segment verwendbar oder reserviert ist)

Sie können das Ergebnis in der Ausgabe von `dmesg` sehen, etwa so:

```
[    0.000000] e820: BIOS-provided physical RAM map:
[    0.000000] BIOS-e820: [mem 0x0000000000000000-0x000000000009fbff] usable
[    0.000000] BIOS-e820: [mem 0x000000000009fc00-0x000000000009ffff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000000f0000-0x00000000000fffff] reserved
[    0.000000] BIOS-e820: [mem 0x0000000000100000-0x000000003ffdffff] usable
[    0.000000] BIOS-e820: [mem 0x000000003ffe0000-0x000000003fffffff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000fffc0000-0x00000000ffffffff] reserved
```

## Initialisierung der Tastatur

Der nächste Schritt ist die Initialisierung der Tastatur mit einem Aufruf der Funktion [`keyboard_init`](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c). Zuerst initialisiert `keyboard_init` die Register mit der Funktion`initregs`. Anschließend wird der Interrupt [0x16](http://www.ctyme.com/intr/rb-1756.htm) aufgerufen, um den Status der Tastatur abzufragen.

```c
    initregs(&ireg);
    ireg.ah = 0x02;     /* Get keyboard status */
    intcall(0x16, &ireg, &oreg);
    boot_params.kbd_status = oreg.al;
```

Danach ruft es erneut [0x16](http://www.ctyme.com/intr/rb-1757.htm) auf, um die Wiederholungsrate und die Verzögerung einzustellen.

```c
    ireg.ax = 0x0305;   /* Set keyboard repeat rate */
    intcall(0x16, &ireg, NULL);
```

## Querying

Die nächsten Schritte sind Abfragen für verschiedene Parameter. Wir werden nicht näher auf diese Fragen eingehen, aber wir werden in späteren Abschnitten darauf zurückkommen. Werfen wir einen kurzen Blick auf diese Funktionen:

Der erste Schritt ist das Abrufen von Informationen zu [Intel SpeedStep](http://en.wikipedia.org/wiki/SpeedStep) durch Aufrufen der Funktion `query_ist`. Es prüft den CPU-Level und ruft bei Korrektheit `0x15` auf um Informationen zu erhalten und speichert das Ergebnis anschließend in `boot_params`.

Als nächstes erhält die Funktion [query_apm_bios](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/apm.c#L21) [Advanced Power Management](http://en.wikipedia.org/wiki/Advanced_Power_Management) Informationen aus dem BIOS. `query_apm_bios` ruft auch die`0x15` BIOS-Unterbrechung auf, aber mit `ah` = `0x53`, um die `APM`-Installation zu überprüfen. Nachdem `0x15` die Ausführung beendet hat, prüfen die`query_apm_bios` -Funktionen die `PM` -Signatur (es muss`0x504d` sein), das Übertragsflag (es muss 0 sein, wenn `APM` unterstützt wird) und den Wert des`cx`-Registers (Wenn es 0x02 ist, wird die Schnittstelle für den geschützten Modus unterstützt).

Als nächstes ruft es wieder `0x15` auf, aber mit `ax = 0x5304`, um die `APM`-Schnittstelle zu trennen und an die 32-Bit-Protected-Mode-Schnittstelle zu connecten. Am Ende wird `boot_params.apm_bios_info` mit Werten gefüllt, die aus dem BIOS stammen.

Beachten Sie, dass `query_apm_bios` nur ausgeführt wird, wenn in der Konfigurationsdatei das Kompilierzeit-Flag `CONFIG_APM` oder `CONFIG_APM_MODULE` gesetzt wurde:

```C
#if defined(CONFIG_APM) || defined(CONFIG_APM_MODULE)
    query_apm_bios();
#endif
```

Die letzte Funktion ist die Funktion [`query_edd`](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/edd.c#L122), mit der Informationen aus dem BIOS zum erweiterten Festplattenlaufwerk abgefragt werden. Schauen wir uns an, wie `query_edd` implementiert ist.

Zunächst liest es die Option [edd](https://github.com/torvalds/linux/blob/v4.16/Documentation/admin-guide/kernel-parameters.rst) von der Kommandozeile des Kernels aus und wenn dies auf `off` gesetzt ist, dann wird ganz einfach `query_edd` zurückgegeben.

Wenn EDD aktiviert ist, geht `query_edd` über BIOS-unterstützte Festplatten und fragt EDD-Informationen in der folgenden Schleife ab:

```C
for (devno = 0x80; devno < 0x80+EDD_MBR_SIG_MAX; devno++) {
    if (!get_edd_info(devno, &ei) && boot_params.eddbuf_entries < EDDMAXNR) {
        memcpy(edp, &ei, sizeof ei);
        edp++;
        boot_params.eddbuf_entries++;
    }
    ...
    ...
    ...
    }
```

Wobei `0x80` die erste Festplatte ist und der Wert des `EDD_MBR_SIG_MAX`-Makros 16 ist. Es sammelt Daten in einem Array in Form von [edd_info](https://github.com/torvalds/linux/blob/v4.16/include/uapi/linux/edd.h) Strukturen. `get_edd_info` prüft, ob EDD vorhanden ist, indem er den `0x13`-Interrupt mit `ah` als`0x41` aufruft. Wenn EDD vorhanden ist, ruft `get_edd_info` erneut den`0x13`-Interrupt auf, aber mit `ah` als `0x48` und `si` enthält die Adresse des Puffers, in dem die EDD-Informationen gespeichert werden.

## Fazit

Dies ist das Ende des zweiten Teils über das Innere des Linux-Kernels. Im nächsten Teil werden die Einstellung des Videomodus und die übrigen Vorbereitungen vor dem Übergang in den geschützten Modus und dem direkten Übergang in den geschützten Modus gezeigt.

Wenn Sie Fragen oder Anregungen haben, schreiben Sie mir einen Kommentar oder senden Sie einen Ping an [twitter](https://twitter.com/0xAX).

**Bitte beachten Sie, dass Englisch nicht meine Muttersprache ist und ich mich für etwaige Unannehmlichkeiten wirklich entschuldige. Wenn Sie Fehler finden, senden Sie mir bitte PR an [linux-insides](https://github.com/0xAX/linux-internals).**

## Links

- [Protected mode](http://en.wikipedia.org/wiki/Protected_mode)
- [Protected mode](http://wiki.osdev.org/Protected_Mode)
- [Long mode](http://en.wikipedia.org/wiki/Long_mode)
- [Nice explanation of CPU Modes with code](http://www.codeproject.com/Articles/45788/The-Real-Protected-Long-mode-assembly-tutorial-for)
- [How to Use Expand Down Segments on Intel 386 and Later CPUs](http://www.sudleyplace.com/dpmione/expanddown.html)
- [earlyprintk documentation](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/earlyprintk.txt)
- [Kernel Parameters](https://github.com/torvalds/linux/blob/v4.16/Documentation/admin-guide/kernel-parameters.rst)
- [Serial console](https://github.com/torvalds/linux/blob/v4.16/Documentation/admin-guide/serial-console.rst)
- [Intel SpeedStep](http://en.wikipedia.org/wiki/SpeedStep)
- [APM](https://en.wikipedia.org/wiki/Advanced_Power_Management)
- [EDD specification](http://www.t13.org/documents/UploadedDocuments/docs2004/d1572r3-EDD3.pdf)
- [TLDP documentation for Linux Boot Process](http://www.tldp.org/HOWTO/Linux-i386-Boot-Code-HOWTO/setup.html) (old)
- [Previous Part](linux-bootstrap-1.md)
