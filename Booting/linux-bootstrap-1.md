# Kernel booting Prozess. Part 1.

## Vom Bootloader zum Kernel

Wenn Sie meine vorherigen [Blogeinträge](https:///0xax.github.io/categories/assembler/) gelesen haben, haben Sie möglicherweise bemerkt, dass ich mich seit einiger Zeit mit Low-Level-Programmierung befasst habe. Ich habe einige Posts über Assembly-Programmierung für `x86_64` Linux geschrieben und zur gleichen Zeit auch begonnen, mich mit dem Linux-Kernel-Quellcode zu beschäftigen.

Ich habe großes Interesse daran zu verstehen, wie einfache Dinge funktionieren, wie Programme auf meinem Computer ausgeführt werden, wie sie sich im Speicher befinden, wie der Kernel Prozesse und Speicher verwaltet, wie der Netzwerkstapel auf einer niedrigen Ebene funktioniert und viele andere Dinge. Deshalb habe ich mich entschlossen, noch eine Reihe weiterer Beiträge zum Linux-Kernel für die **x86_64**-Architektur zu verfassen.

Beachten Sie, dass ich kein professioneller Kernel-Hacker bin und keinen Code für den Kernel bei der Arbeit schreibe. Es ist nur ein Hobby. Ich mag nur Low-Level-Sachen, und es ist interessant für mich zu sehen, wie diese Dinge funktionieren. Wenn Sie also etwas Verwirrendes bemerken oder Fragen/Anmerkungen haben, senden Sie mir ein Ping-Signal auf Twitter [0xAX](https://twitter.com/0xAX), eine [E-Mail](anotherworldofworld@gmail.com) oder erstelle ein [Issue](https://github.com/0xAX/linux-insides/issues/new). Ich schätze es.

Alle Beiträge können auch unter [github repo](https://github.com/0xAX/linux-insides) abgerufen werden. Wenn Sie Probleme mit meinem Englisch oder dem Inhalt der Beiträge haben, können Sie gerne eine Pull-Anfrage senden.

_Beachten Sie, dass dies keine offizielle Dokumentation ist, sondern nur Lernen und Teilen von Wissen._

**Vorausgesetztes Wissen**

- Vestehen von C code
- Vestehen von assembly code (AT&T syntax)

Wie auch immer, wenn Sie gerade erst anfangen, solche Tools zu erlernen, werde ich versuchen, einige Teile in diesem und den folgenden Beiträgen zu erklären. Okay, das ist das Ende der einfachen Einführung, und jetzt können wir anfangen, uns mit dem Linux-Kernel und dem Low-Level-Zeug zu beschäftigen.

Ich habe angefangen, dieses Buch zur Zeit des Linux-Kernels `3.18` zu schreiben und viele Dinge könnten sich seitdem geändert haben. Bei Änderungen werde ich die Beiträge entsprechend aktualisieren.

## Der magische Power Button, was passiert als nächstes?

Obwohl es sich um eine Reihe von Beiträgen zum Linux-Kernel handelt, werden wir nicht direkt mit dem Kernel-Code beginnen - zumindest nicht in diesem Abschnitt. Sobald Sie den magischen Ein-/Ausschalter Ihres Laptops oder Desktop-Computers drücken, funktioniert es. Das Motherboard sendet ein Signal an das [Netzteil](https://en.wikipedia.org/wiki/Power_supply). Nach dem Empfang des Signals versorgt das Netzteil den Computer mit der richtigen Menge an Strom. Sobald das Motherboard das [Power Good Signal](https://en.wikipedia.org/wiki/Power_good_signal) empfängt, versucht es die CPU zu starten. Die CPU setzt alle verbleibenden Daten in ihren Registern zurück und legt für jedes von ihnen vordefinierte Werte fest.

Die CPU [80386](https://en.wikipedia.org/wiki/Intel_80386) und spätere CPUs definieren die folgenden vordefinierten Daten in CPU-Registern, nachdem der Computer zurückgesetzt wurde:

```
IP          0xfff0
CS selector 0xf000
CS base     0xffff0000
```

Der Prozessor beginnt im [Real-Modus](https://en.wikipedia.org/wiki/Real_mode) zu arbeiten. Lassen Sie uns ein wenig zurückblättern und versuchen, die [Speichersegmentierung](https://en.wikipedia.org/wiki/Memory_segmentation) in diesem Modus zu verstehen. Der Real-Modus wird von allen x86-kompatiblen Prozessoren unterstützt, von der [8086](https://en.wikipedia.org/wiki/Intel_8086)-CPU bis hin zu den modernen Intel 64-Bit-CPUs. Der `8086`-Prozessor verfügt über einen 20-Bit-Adressbus, was bedeutet, dass er mit einem `0-0xFFFFF` oder `1 megabyte`-Adressraum arbeiten kann. Er hat aber nur `16-bit`-Register, die eine maximale Adresse von `2^16 - 1` oder `0xffff` (64 Kilobyte) haben.

[Memory Segmentation](https://en.wikipedia.org/wiki/Memory_segmentation) wird verwendet, um den gesamten verfügbaren Adressraum zu nutzen. Der gesamte Speicher ist in kleine Segmente mit fester Größe von `65536` Byte (64 KB) unterteilt. Da wir mit 16-Bit-Registern keinen Speicher über `64 KB` adressieren können, wurde eine alternative Methode entwickelt.

Eine Adresse besteht aus zwei Teilen: einem Segmentselektor, der eine Basisadresse hat, und einem Versatz von dieser Basisadresse. Im Real-Modus lautet die zugehörige Basisadresse eines Segmentwählers `Segment Selector * 16`. Um eine physikalische Adresse im Speicher zu erhalten, müssen wir den Segment-Selektor-Teil mit "16" multiplizieren und den Versatz hinzufügen:

```
PhysicalAddress = Segment Selector * 16 + Versatz
```

Wenn beispielsweise `CS:IP` `0x2000:0x0010` ist, lautet die entsprechende physikalische Adresse:

```python
>>> hex((0x2000 << 4) + 0x0010)
'0x20010'
```

Wenn wir jedoch die größte Segmentauswahl und den größten Versatz `0xffff:0xffff` verwenden, lautet die resultierende Adresse:

```python
>>> hex((0xffff << 4) + 0xffff)
'0x10ffef'
```

Das sind `65520` Bytes nach dem ersten Megabyte. Da im Real-Modus nur auf einen Megabyte zugegriffen werden kann, wird `0x10ffef` zu `0x00ffef` und die [A20-Zeile](https://en.wikipedia.org/wiki/A20_line) wird deaktiviert.

Ok, jetzt wissen wir ein wenig über den Real-Modus und die Speicheradressierung in diesem Modus. Kommen wir zurück zur Diskussion der Registerwerte nach dem Zurücksetzen.

Das `CS` -Register besteht aus zwei Teilen: Dem Selektor für sichtbare Segmente und der versteckten Basisadresse. Während die Basisadresse normalerweise durch Multiplizieren des Segmentselektorwerts mit 16 gebildet wird, wird während eines Hardware-Resets der Segmentselektor im CS-Register mit `0xf000` und die Basisadresse mit `0xffff0000` geladen; Der Prozessor verwendet diese spezielle Basisadresse, bis `CS` geändert wird.

Die Startadresse wird gebildet, indem die Basisadresse zum Wert im EIP-Register addiert wird:

```python
>>> 0xffff0000 + 0xfff0
'0xfffffff0'
```

Wir erhalten `0xfffffff0`, was 16 Bytes weniger als 4 GB ist. Dieser Punkt wird als [Vektor zurücksetzen](https://en.wikipedia.org/wiki/Reset_vector) bezeichnet. Dies ist der Speicherort, an dem die CPU nach dem Zurücksetzen den ersten auszuführenden Befehl erwartet. Es enthält eine Anweisung [jump](https://en.wikipedia.org/wiki/JMP_%28x86_instruction%29) (`jmp`), die normalerweise auf den BIOS-Einstiegspunkt verweist. Wenn wir uns zum Beispiel den Quellcode von [coreboot](https://www.coreboot.org/) ansehen (`src/cpu/x86/16bit/reset16.inc`), werden wir sehen:

```assembly
    .section ".reset", "ax", %progbits
    .code16
.globl	_start
_start:
    .byte  0xe9
    .int   _start16bit - ( . + 2 )
    ...
```

Hier sehen wir den Befehl `jmp` [opcode](http://ref.x86asm.net/coder32.html#xE9), welcher `0xe9` ist und auf die Zieladresse `_start16bit - ( . + 2)` zeigt.

Wir können auch sehen, dass der `reset` Abschnitt `16` Bytes hat und so kompiliert ist, dass er von der Adresse `0xfffffff0` (`src/cpu/x86/16bit/reset16.ld`) beginnt:

```
SECTIONS {
    /* Trigger an error if I have an unuseable start address */
    _bogus = ASSERT(_start16bit >= 0xffff0000, "_start16bit too low. Please report.");
    _ROMTOP = 0xfffffff0;
    . = _ROMTOP;
    .reset . : {
        *(.reset);
        . = 15;
        BYTE(0x00);
    }
}
```

Nun startet das BIOS. Nach der Initialisierung und Überprüfung der Hardware muss das BIOS ein bootfähiges Gerät finden. In der BIOS-Konfiguration ist eine Startreihenfolge gespeichert, die steuert, von welchen Geräten das BIOS zu starten versucht. Beim Versuch, von einer Festplatte zu booten, versucht das BIOS, einen Bootsektor zu finden. Auf Festplatten, die mit einem [MBR-Partitionslayout](https://en.wikipedia.org/wiki/Master_boot_record) partitioniert sind, wird der Bootsektor in den ersten `446` Bytes des ersten Sektors gespeichert, wobei jeder Sektor `512` Bytes beträgt. Die letzten zwei Bytes des ersten Sektors sind `0x55` und `0xaa`, was dem BIOS signalisiert, dass dieses Gerät bootfähig ist.

Zum Beispiel:

```assembly
;
; Note: this example is written in Intel Assembly syntax
;
[BITS 16]

boot:
    mov al, '!'
    mov ah, 0x0e
    mov bh, 0x00
    mov bl, 0x07

    int 0x10
    jmp $

times 510-($-$$) db 0

db 0x55
db 0xaa
```

Bauen und starten Sie dies mit:

```
nasm -f bin boot.nasm && qemu-system-x86_64 boot
```

Dadurch wird [QEMU](http://qemu.org) angewiesen, die soeben erstellte `boot`-Binärdatei als Disk-Image zu verwenden. Da die durch den obigen Assembly-Code erzeugte Binärdatei die Anforderungen des Boot-Sektors erfüllt (der Ursprung ist auf `0x7c00` gesetzt und wir beenden sie mit der magischen Sequenz), behandelt QEMU die Binärdatei als Master Boot Record (MBR) von einem Disk-Image.

Du wirst folgendes sehen:

![Einfacher Bootloader, der nur druckt `!`](http://oi60.tinypic.com/2qbwup0.jpg)

In diesem Beispiel können wir sehen, dass der Code im `16-bit`-Bit-Real-Modus ausgeführt wird und bei `0x7c00` im Speicher beginnt. Nach dem Start wird der Interrupt [0x10](http://www.ctyme.com/intr/rb-0106.htm) aufgerufen, der nur das Symbol `!` Ausgibt. es füllt die verbleibenden `510` Bytes mit Nullen und endet mit den zwei magischen Bytes `0xaa` und `0x55`.

Mit dem Hilfsprogramm `objdump` können Sie sich einen binären Speicherauszug davon anzeigen lassen:

```
nasm -f bin boot.nasm
objdump -D -b binary -mi386 -Maddr16,data16,intel boot
```

Ein realer Bootsektor enthält Code zum Fortsetzen des Bootvorgangs und eine Partitionstabelle anstelle einer Reihe von Nullen und eines Ausrufezeichens :) Ab diesem Zeitpunkt übergibt das BIOS die Steuerung an den Bootloader.

**HINWEIS**: Wie oben erläutert, befindet sich die CPU im Real-Modus. Im Real-Modus wird die physikalische Adresse im Speicher wie folgt berechnet:

```
PhysicalAddress = Segment Selector * 16 + Versatz
```

genau wie oben erklärt. Wir haben nur 16-Bit-Allzweckregister; Der Maximalwert eines 16-Bit-Registers ist `0xffff`. Wenn wir also die größten Werte annehmen, ist das Ergebnis:

```python
>>> hex((0xffff * 16) + 0xffff)
'0x10ffef'
```

wobei `0x10ffef` gleich `1MB + 64KB - 16b` ist. Ein Prozessor [8086](https://en.wikipedia.org/wiki/Intel_8086) (der erste Prozessor mit Real-Modus) hat dagegen eine 20-Bit-Adresszeile. Da `2^20 = 1048576` 1 MB beträgt, bedeutet dies, dass der tatsächlich verfügbare Speicher 1 MB beträgt.

Im Allgemeinen sieht die Speicherzuordnung des Real-Modus wie folgt aus:

```
0x00000000 - 0x000003FF - Real Mode Interrupt Vector Table
0x00000400 - 0x000004FF - BIOS Data Area
0x00000500 - 0x00007BFF - Unused
0x00007C00 - 0x00007DFF - Our Bootloader
0x00007E00 - 0x0009FFFF - Unused
0x000A0000 - 0x000BFFFF - Video RAM (VRAM) Memory
0x000B0000 - 0x000B7777 - Monochrome Video Memory
0x000B8000 - 0x000BFFFF - Color Video Memory
0x000C0000 - 0x000C7FFF - Video ROM BIOS
0x000C8000 - 0x000EFFFF - BIOS Shadow Area
0x000F0000 - 0x000FFFFF - System BIOS
```

Am Anfang dieses Beitrags habe ich geschrieben, dass sich der erste von der CPU ausgeführte Befehl unter der Adresse `0xFFFFFFF0` befindet, die viel größer als `0xFFFFF` (1 MB) ist. Wie kann die CPU im Real-Modus auf diese Adresse zugreifen? Die Antwort finden Sie in der Dokumentation zu [coreboot](https://www.coreboot.org/Developer_Manual/Memory_map):

```
0xFFFE_0000 - 0xFFFF_FFFF: 128 kilobyte ROM mapped into address space
```

Zu Beginn der Ausführung befindet sich das BIOS nicht im RAM, sondern im ROM.

## Bootloader

Es gibt eine Reihe von Bootloadern, die Linux booten können, z. B. [GRUB 2](https://www.gnu.org/software/grub/) und [syslinux](http://www.syslinux.org/wiki/index.php/The_Syslinux_Project). Der Linux-Kernel verfügt über ein [Boot-Protokoll](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/boot.txt), das die Anforderungen für einen Bootloader zur Implementierung der Linux-Unterstützung festlegt. In diesem Beispiel wird GRUB 2 beschrieben.

Nachdem das `BIOS` ein Startgerät ausgewählt und die Steuerung auf den Startsektorcode übertragen hat, wird die Ausführung von [boot.img](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/boot/i386/pc/boot.S;hb=HEAD) aus gestartet. Dieser Code ist aufgrund des begrenzten verfügbaren Speicherplatzes sehr einfach und enthält einen Zeiger, mit dem Sie zum Speicherort des GRUB 2-Kernimages springen können. Das Kernimage beginnt mit [diskboot.img](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/boot/i386/pc/diskboot.S;hb=HEAD), der normalerweise unmittelbar nach dem ersten Sektor in dem nicht verwendeten Raum vor der ersten Partition gespeichert wird. Der obige Code lädt den Rest des Core-Image, das den GRUB 2-Kernel und die Treiber für den Umgang mit Dateisystemen enthält, in den Arbeitsspeicher. Nach dem Laden des restlichen Core-Images wird [grub_main](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/kern/main.c) ausgeführt.

Die Funktion `grub_main` initialisiert die Konsole, ruft die Basisadresse für Module ab, legt das Root-Gerät fest, lädt/analysiert die Grub-Konfigurationsdatei, lädt Module usw. Am Ende der Ausführung versetzt die Funktion `grub_main` Grub in den normalen Modus. Die Funktion `grub_normal_execute` (aus der Quellcodedatei "grub-core/normal/main.c") schließt die letzten Vorbereitungen ab und zeigt ein Menü zur Auswahl eines Betriebssystems an. Wenn wir einen der Menüeinträge grub auswählen, wird die Funktion `grub_menu_execute_entry` ausgeführt, wobei der grub Befehl `boot` ausgeführt und das ausgewählte Betriebssystem gebootet wird.

Wie wir im Kernel-Boot-Protokoll lesen können, muss der Bootloader einige Felder des Kernel-Setup-Headers lesen und ausfüllen, der mit dem Versatz `0x01f1` vom Kernel-Setup-Code beginnt. Sie können den Wert dieses Versatzes im Boot [Linker-Skript](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld) überprüfen. Der Kernel-Header [arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S) beginnt mit:

```assembly
    .globl hdr
hdr:
    setup_sects: .byte 0
    root_flags:  .word ROOT_RDONLY
    syssize:     .long 0
    ram_size:    .word 0
    vid_mode:    .word SVGA_MODE
    root_dev:    .word 0
    boot_flag:   .word 0xAA55
```

Der Bootloader muss dies und den Rest der Header ausfüllen (die im Linux-Bootprotokoll nur als "write" gekennzeichnet sind, wie in [diesem Beispiel](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/boot.txt#L354)) mit Werten, die es entweder von der Kommandozeile erhalten oder beim Booten berechnet hat. (Wir werden jetzt nicht alle Beschreibungen und Erklärungen für alle Felder des Kernel-Setup-Headers durchgehen, aber wir werden dies tun, wenn wir besprechen, wie der Kernel sie verwendet. Eine Beschreibung aller Felder finden Sie im [Boot-Protokoll](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/boot.txt#L156).)

Wie wir im Kernel-Boot-Protokoll sehen können, wird der Speicher nach dem Laden des Kernels wie folgt zugeordnet:

```shell
         | Protected-mode kernel  |
100000   +------------------------+
         | I/O memory hole        |
0A0000   +------------------------+
         | Reserved for BIOS      | Leave as much as possible unused
         ~                        ~
         | Command line           | (Can also be below the X+10000 mark)
X+10000  +------------------------+
         | Stack/heap             | For use by the kernel real-mode code.
X+08000  +------------------------+
         | Kernel setup           | The kernel real-mode code.
         | Kernel boot sector     | The kernel legacy boot sector.
       X +------------------------+
         | Boot loader            | <- Boot sector entry point 0x7C00
001000   +------------------------+
         | Reserved for MBR/BIOS  |
000800   +------------------------+
         | Typically used by MBR  |
000600   +------------------------+
         | BIOS use only          |
000000   +------------------------+

```

Wenn der Bootloader also die Kontrolle an den Kernel überträgt, beginnt er bei:

```
X + sizeof(KernelBootSector) + 1
```

Dabei ist `X` die Adresse des Kernel-Boot-Sektors, der geladen wird. In meinem Fall ist `X` `0x10000`, wie wir in einem Speicherauszug sehen können:

![Erste Adresse des Kernels](http://oi57.tinypic.com/16bkco2.jpg)

Der Bootloader hat nun den Linux-Kernel in den Speicher geladen, die Header-Felder ausgefüllt und ist dann zur entsprechenden Speicheradresse gesprungen. Wir können jetzt direkt zum Kernel-Setup-Code wechseln.

## Der Beginn der Kernel-Setup-Phase

Endlich sind wir im Kernel! Technisch gesehen ist der Kernel noch nicht gelaufen. Zunächst muss der Kernel-Setup-Teil Dinge wie den Dekomprimierer und einige mit der Speicherverwaltung zusammenhängende Dinge konfigurieren, um nur wenige zu nennen. Nachdem all diese Dinge erledigt sind, dekomprimiert der Kernel-Setup-Teil den eigentlichen Kernel und springt dorthin. Die Ausführung des Setup-Teils beginnt bei [arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S) am [\_start](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L292) Symbol.

Auf den ersten Blick mag es etwas seltsam aussehen, da mehrere Anweisungen davor stehen. Vor langer Zeit hatte der Linux-Kernel einen eigenen Bootloader. Jetzt jedoch, wenn Sie zum Beispiel folgendes ausführen,

```
qemu-system-x86_64 vmlinuz-3.18-generic
```

so werden Sie folgendes sehen:

![Versuchen Sie es mit vmlinuz in qemu](http://oi60.tinypic.com/r02xkz.jpg)

Tatsächlich beginnt die Datei `header.S` mit der magischen Nummer [MZ](https://en.wikipedia.org/wiki/DOS_MZ_executable) (siehe Abbildung oben), der angezeigten Fehlermeldung und anschließend dem [PE](https://en.wikipedia.org/wiki/Portable_Executable)-Header:

```assembly
#ifdef CONFIG_EFI_STUB
# "MZ", MS-DOS header
.byte 0x4d
.byte 0x5a
#endif
...
...
...
pe_header:
    .ascii "PE"
    .word 0
```

Dies ist erforderlich, um ein Betriebssystem mit Unterstützung für [UEFI](https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface) zu laden. Wir werden uns im Moment nicht mit seiner internen Funktionsweise befassen, sondern später in den nächsten Kapiteln darauf eingehen.

Der eigentliche Einstiegspunkt für das Kernel-Setup ist:

```assembly
// header.S line 292
.globl _start
_start:
```

Der Bootloader (grub2 und andere) kennt diesen Punkt (mit einem Versatz von `0x200` von `MZ`) und springt direkt dorthin, obwohl `header.S` im Abschnitt`.bstext` beginnt, was eine Fehlermeldung ausgibt:

```
//
// arch/x86/boot/setup.ld
//
. = 0;                    // current position
.bstext : { *(.bstext) }  // put .bstext section to position 0
.bsdata : { *(.bsdata) }
```

Der Einstiegspunkt für das Kernel-Setup ist:

```assembly
    .globl _start
_start:
    .byte  0xeb
    .byte  start_of_setup-1f
1:
    //
    // rest of the header
    //
```

Hier sehen wir einen `jmp`-Anweisungs-Opcode (`0xeb`), der zum `start_of_setup-1f`-Punkt springt. In der `Nf`-Notation bezieht sich `2f` beispielsweise auf die lokale Bezeichnung `2:`; In unserem Fall ist es die Bezeichnung `1`, die direkt nach dem Sprung vorhanden ist und den Rest des Setups [header](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/boot.txt#L156) enthält. Direkt nach dem Setup-Header sehen wir den Abschnitt `.entrytext`, der mit dem Label `start_of_setup` beginnt.

Dies ist der erste Code, der tatsächlich ausgeführt wird (abgesehen von den vorherigen Sprunganweisungen natürlich). Nachdem der Kernel-Setup-Teil die Steuerung vom Bootloader erhalten hat, befindet sich der erste Befehl `jmp` am `0x200`-Versatz vom Start des Kernel-Real-Modus, d. H. Nach den ersten 512 Bytes. Dies ist sowohl im Linux-Kernel-Boot-Protokoll als auch im grub2-Quellcode zu sehen:

```C
segment = grub_linux_real_target >> 4;
state.gs = state.fs = state.es = state.ds = state.ss = segment;
state.cs = segment + 0x20;
```

In meinem Fall wird der Kernel unter der physikalischen Adresse `0x10000` geladen. Dies bedeutet, dass die Segmentregister nach dem Start des Kernel-Setups die folgenden Werte aufweisen:

```
gs = fs = es = ds = ss = 0x1000
cs = 0x1020
```

Nach dem Sprung zu `start_of_setup` muss der Kernel folgende Schritte ausführen:

- Stellen Sie sicher, dass alle Segmentregisterwerte gleich sind
- Richten Sie bei Bedarf einen korrekten Stack ein
- Einrichten von [bss](https://en.wikipedia.org/wiki/.bss)
- Wechseln Sie zum C-Code in [arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c)

Schauen wir uns die Implementierung an.

## Ausrichten der Segmentregister

Zunächst stellt der Kernel sicher, dass die Segmentregister `ds` und `es` auf dieselbe Adresse verweisen. Als nächstes wird das Richtungsflag mit der Anweisung `cld` gelöscht:

```assembly
    movw    %ds, %ax
    movw    %ax, %es
    cld
```

Wie ich bereits erwähnt habe, lädt `grub2` den Kernel-Setup-Code standardmäßig unter der Adresse`0x10000` und `cs` unter der Adresse`0x1020`, da die Ausführung nicht am Anfang der Datei beginnt, sondern beim folgenden Sprung:

```assembly
_start:
    .byte 0xeb
    .byte start_of_setup-1f
```

Dies ist ein Abstand von `512` Byte zu [4d 5a](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L46). Wir müssen außerdem `cs` von `0x1020` auf `0x1000` sowie alle anderen Segmentregister ausrichten. Danach richten wir den Stack ein:

```assembly
    pushw   %ds
    pushw   $6f
    lretw
```

Hierdurch wird der Wert von `ds` auf den Stack verschoben, gefolgt von der Adresse vom Label [6](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L602) und führt den Befehl `lretw` aus. Wenn der Befehl `lretw` aufgerufen wird, lädt er die Adresse des Labels `6` in das Register [Anweisungszeiger](https://en.wikipedia.org/wiki/Program_counter) und lädt `cs` mit dem Wert von`ds`. Danach haben `ds` und `cs` die gleichen Werte.

## Stack Setup

Fast der gesamte Setup-Code dient zur Vorbereitung der C-Sprachumgebung im Real-Modus. Der nächste [Schritt](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L575) prüft den Wert des `ss`-Registers und richtet einen korrekten Stack ein wenn `ss` falsch ist:

```assembly
    movw    %ss, %dx
    cmpw    %ax, %dx
    movw    %sp, %dx
    je      2f
```

Dies kann zu 3 verschiedenen Szenarien führen:

- `ss` hat einen gültigen Wert `0x1000` (wie alle anderen Segmentregister neben `cs`)
- `ss` ist ungültig und das `CAN_USE_HEAP` Flag ist gesetzt (siehe unten)
- `ss` ist ungültig und das `CAN_USE_HEAP` Flag ist nicht gesetzt (siehe unten)

Sehen wir uns nacheinander alle drei dieser Szenarien an:

- `ss` hat eine korrekte Adresse (`0x1000`). In diesem Fall gehen wir zu Label [2](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L589):

```assembly
2:  andw    $~3, %dx
    jnz     3f
    movw    $0xfffc, %dx
3:  movw    %ax, %ss
    movzwl  %dx, %esp
    sti
```

Hier setzen wir die Ausrichtung von `dx` (die den vom Bootloader angegebenen Wert von`sp` enthält) auf `4` Bytes und prüfen, ob er Null ist. Wenn dies der Fall ist, setzen wir `dx` auf `0xfffc` (die letzte 4-Byte-ausgerichtete Adresse in einem 64-KB-Segment). Wenn dieser nicht Null ist, verwenden wir weiterhin den vom Bootloader angegebenen Wert von `sp` (in meinem Fall 0xf7f4). Danach setzen wir den Wert von `ax` in `ss`, was bedeutet, dass `ss` den Wert `0x1000` enthält. Wir haben jetzt einen korrekten Stack:

![stack](http://oi58.tinypic.com/16iwcis.jpg)

- Im zweiten Szenario (`ss` != `ds`). Zuerst setzen wir den Wert von [\_end](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld) (die Adresse des Endes des Setup-Codes) ) in `dx` und überprüfe das`loadflags`-Headerfeld mit der `testb`-Anweisung, um zu sehen, ob wir den Heap benutzen können. [loadflags](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L320) ist ein Bitmasken-Header, der wie folgt definiert ist:

```C
#define LOADED_HIGH     (1<<0)
#define QUIET_FLAG      (1<<5)
#define KEEP_SEGMENTS   (1<<6)
#define CAN_USE_HEAP    (1<<7)
```

und, wie wir im Boot-Protokoll lesen können:

```
Field name: loadflags

  This field is a bitmask.

  Bit 7 (write): CAN_USE_HEAP
    Set this bit to 1 to indicate that the value entered in the
    heap_end_ptr is valid.  If this field is clear, some setup code
    functionality will be disabled.
```

Wenn das `CAN_USE_HEAP`-Bit gesetzt ist, setzen wir `heap_end_ptr` in `dx` (was auf `_end` zeigt) und fügen `STACK_SIZE` (die minimale Stapelgröße, `1024` Bytes) hinzu. Danach, wenn `dx` nicht getragen wird (es wird nicht getragen, `dx = _end + 1024`), springe zu Label `2` (wie im vorherigen Fall) und mache einen korrekten Stapel.

![stack](http://oi62.tinypic.com/dr7b5w.jpg)

- Wenn `CAN_USE_HEAP` nicht gesetzt ist, verwenden wir nur einen minimalen Stack von `_end` bis `_end + STACK_SIZE`:

![minimal stack](http://oi60.tinypic.com/28w051y.jpg)

## BSS Setup

Die letzten beiden Schritte, die ausgeführt werden müssen, bevor wir zum Haupt-C-Code springen können, sind das Einrichten des Bereichs [BSS](https://en.wikipedia.org/wiki/.bss) und das Überprüfen der "magischen" Signatur. Zuerst zur Überprüfung der Signatur:

```assembly
    cmpl    $0x5a5aaa55, setup_sig
    jne     setup_bad
```

Dies vergleicht einfach die [setup_sig](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld) mit der magischen Zahl `0x5a5aaa55`. Wenn sie nicht gleich sind, wird ein schwerwiegender Fehler gemeldet.

Wenn die magische Zahl übereinstimmt und wir wissen, dass wir einen Satz korrekter Segmentregister und einen Stapel haben, müssen wir nur den BSS-Abschnitt einrichten, bevor wir in den C-Code springen.

Im BSS-Bereich werden statisch zugewiesene, nicht initialisierte Daten gespeichert. Linux achtet sorgfältig darauf, dass dieser Speicherbereich zuerst mit dem folgenden Code auf Null gesetzt wird:

```assembly
    movw    $__bss_start, %di
    movw    $_end+3, %cx
    xorl    %eax, %eax
    subw    %di, %cx
    shrw    $2, %cx
    rep; stosl
```

Zunächst wird die Adresse [\_\_bss_start](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld) in `di` verschoben. Als nächstes wird die Adresse `_end + 3` (+3 - entspricht 4 Bytes) in `cx` verschoben. Das `eax`-Register wird gelöscht (unter Verwendung einer `xor`-Anweisung), und die BSS-Abschnittsgröße (`cx`-`di`) wird berechnet und in `cx` gestellt. Dann wird `cx` durch vier geteilt (die Größe eines "Wortes"), und der Befehl `stosl` wird wiederholt verwendet, wobei der Wert von `eax` (Null) automatisch in der von `di` angegebenen Adresse gespeichert wird Erhöhen Sie `di` um vier und wiederholt den Vorgang solange bis `cx` Null erreicht. Der Nettoeffekt dieses Codes besteht darin, dass Nullen durch alle Wörter im Speicher von `__bss_start` bis `_end` geschrieben werden:

![bss](http://oi59.tinypic.com/29m2eyr.jpg)

## Zu main springen

Das ist alles - wir haben den Stack und BSS, also können wir zur C-Funktion `main()` springen:

```assembly
    calll main
```

Die Funktion `main()` befindet sich in [arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c). Im nächsten Teil können Sie nachlesen, was dies bewirkt.

## Fazit

Dies ist das Ende des ersten Teils über das Innere des Linux-Kernels. Wenn Sie Fragen oder Vorschläge haben, senden Sie mir ein Ping-Signal auf Twitter [0xAX](https://twitter.com/0xAX), senden Sie mir eine [E-Mail](anotherworldofworld@gmail.com) oder erstellen Sie einfach ein [Issue](https://github.com/0xAX/linux-internals/issues/new). Im nächsten Teil werden wir den ersten C-Code sehen, der im Linux-Kernel-Setup ausgeführt wird, die Implementierung von Speicherroutinen wie `memset`,`memcpy`, `earlyprintk`, frühe Implementierung und Initialisierung der Konsole und vieles mehr.

**Bitte beachten Sie, dass Englisch nicht meine Muttersprache ist und ich mich für etwaige Unannehmlichkeiten wirklich entschuldige. Wenn Sie Fehler finden, senden Sie mir bitte PR an [linux-insides](https://github.com/0xAX/linux-internals).**

## Links

- [Intel 80386 programmer's reference manual 1986](http://css.csail.mit.edu/6.858/2014/readings/i386.pdf)
- [Minimal Boot Loader for Intel® Architecture](https://www.cs.cmu.edu/~410/doc/minimal_boot.pdf)
- [Minimal Boot Loader in Assembler with comments](https://github.com/Stefan20162016/linux-insides-code/blob/master/bootloader.asm)
- [8086](https://en.wikipedia.org/wiki/Intel_8086)
- [80386](https://en.wikipedia.org/wiki/Intel_80386)
- [Reset vector](https://en.wikipedia.org/wiki/Reset_vector)
- [Real mode](https://en.wikipedia.org/wiki/Real_mode)
- [Linux kernel boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt)
- [coreboot developer manual](https://www.coreboot.org/Developer_Manual)
- [Ralf Brown's Interrupt List](http://www.ctyme.com/intr/int.htm)
- [Power supply](https://en.wikipedia.org/wiki/Power_supply)
- [Power good signal](https://en.wikipedia.org/wiki/Power_good_signal)
