# Kernel-Bootvorgang. Teil 3.

## Initialisierung des Videomodus und Übergang in den geschützten Modus

Dies ist der dritte Teil der `Kernel booting process`-Serie. Im vorherigen [part](linux-bootstrap-2.md#kernel-booting-process-part-2) haben wir direkt vor dem Aufruf der `set_video`-Routine von [main.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c) angehalten.

In diesem Teil betrachten wir:

- Initialisierung des Videomodus im Kernel-Setup-Code,
- die Vorbereitungen vor dem Wechsel in den geschützten Modus,
- den Übergang in den geschützten Modus

**HINWEIS** Wenn Sie nichts über den geschützten Modus wissen, finden Sie einige Informationen dazu im vorherigen [Teil](linux-bootstrap-2.md#protected-mode). Es gibt auch ein paar [links](linux-bootstrap-2.md#links), die Ihnen helfen können.

Wie ich oben geschrieben habe, starten wir mit der Funktion `set_video`, die in der Quellcode-Datei [arch/x86/boot/video.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/video.c) definiert ist. Wir können sehen, dass es zunächst damit beginnt den Videomodus aus der `boot_params.hdr`-Struktur abrufen:

```C
u16 mode = boot_params.hdr.vid_mode;
```

was wir in der Funktion `copy_boot_params` schon mal gehabt haben (ihr könnt es im vorherigen Beitrag nachlesen). `vid_mode` ist ein Pflichtfeld, das vom Bootloader ausgefüllt wird. Informationen dazu finden Sie im `Boot-Protokoll` des Kernels:

```
Offset	Proto	Name		Meaning
/Size
01FA/2	ALL	    vid_mode	Video mode control
```

Wie wir aus dem Linux-Kernel-Boot-Protokoll herauslesen können:

```
vga=<mode>
	<mode> here is either an integer (in C notation, either
	decimal, octal, or hexadecimal) or one of the strings
	"normal" (meaning 0xFFFF), "ext" (meaning 0xFFFE) or "ask"
	(meaning 0xFFFD).  This value should be entered into the
	vid_mode field, as it is used by the kernel before the command
	line is parsed.
```

Daher können wir der Konfigurationsdatei von grub (oder einem anderen Bootloader) die Option `vga` hinzufügen, die an die Kernel-Befehlszeile übergeben wird. Diese Option kann unterschiedliche Werte haben, wie in der Beschreibung angegeben. Beispielsweise kann es sich um eine Ganzzahl `0xFFFD` oder `ask` handeln. Wenn Sie `ask` an `vga` übergeben, wird ein Menü wie das folgende angezeigt:

![Einstellungsmenü für den Videomodus](http://oi59.tinypic.com/ejcz81.jpg)

Daraufhin werden Sie aufgefordert, einen Videomodus auszuwählen. Wir werden uns die Implementierung ansehen, aber bevor wir uns mit der Implementierung befassen, müssen wir uns noch einige andere Dinge ansehen.

## Kernel-Datentypen

Früher sahen wir Definitionen verschiedener Datentypen wie `u16` usw. im Kernel-Setup-Code. Schauen wir uns ein paar Datentypen an, die vom Kernel bereitgestellt werden:

| Type | char | short | int | long | u8  | u16 | u32 | u64 |
| ---- | ---- | ----- | --- | ---- | --- | --- | --- | --- |
| Size | 1    | 2     | 4   | 8    | 1   | 2   | 4   | 8   |

Wenn Sie den Quellcode des Kernels lesen, werden Sie diese sehr oft sehen und es wird gut sein, sich an sie zu erinnern.

## Heap API

Nachdem wir `vid_mode` von `boot_params.hdr` in der `set_video`-Funktion erhalten haben, können wir den Aufruf der `RESET_HEAP`-Funktion sehen. `RESET_HEAP` ist ein Makro, das in der Header-Datei [arch/x86/boot/boot.h](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/boot.h) definiert ist.

Dieses Makro ist definiert als:

```C
#define RESET_HEAP() ((void *)( HEAP = _end ))
```

Wenn Sie den zweiten Teil gelesen haben, werden Sie sich daran erinnern, dass wir den Heap mit der Funktion [`init_heap`](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c) initialisiert haben. Wir haben ein paar Utility-Makros und Funktionen für die Verwaltung des Heaps, die in der Header-Datei `arch/x86/boot/boot.h` definiert sind.

Diese sind:

```C
#define RESET_HEAP()
```

Wie wir oben gesehen haben, wird der Heap zurückgesetzt, indem die `HEAP`-Variable auf `_end` gesetzt wird, wobei `_end` nur `extern char _end[];` ist.

Als nächstes kommt das Makro `GET_HEAP`:

```C
#define GET_HEAP(type, n) \
	((type *)__get_heap(sizeof(type),__alignof__(type),(n)))
```

für die Heap-Zuordnung. Es ruft die interne Funktion `__get_heap` mit 3 Parametern auf:

- die Größe des zuzuordnenden Datentyps
- `__alignof __ (type)` gibt an, wie Variablen dieses Typs ausgerichtet werden sollen
- `n` gibt an, wie viele Elemente zugeordnet werden sollen

Die Implementierung von `**get_heap` ist:

```C
static inline char *__get_heap(size_t s, size_t a, size_t n)
{
	char *tmp;

	HEAP = (char *)(((size_t)HEAP+(a-1)) & ~(a-1));
	tmp = HEAP;
	HEAP += s*n;
	return tmp;
}
```

wir werden später seine Verwendung sehen, so etwas wie:

```C
saved.data = GET_HEAP(u16, saved.x * saved.y);
```

Versuchen wir zu verstehen, wie `__get_heap` funktioniert. Wir können hier sehen, dass `HEAP` (das gleich `_end` nach `RESET_HEAP()` ist) die Adresse des ausgerichteten Speichers gemäß dem `a`-Parameter zugewiesen bekommt. Danach speichern wir die Speicheradresse von `HEAP` in die Variable `tmp`, verschieben `HEAP` an das Ende des zugewiesenen Blocks und geben `tmp` zurück, welches die Startadresse des zugewiesenen Speichers ist.

Und die letzte Funktion ist:

```C
static inline bool heap_free(size_t n)
{
	return (int)(heap_end - HEAP) >= (int)n;
}
```

Dieser subtrahiert den Wert des `HEAP`-Zeigers vom`heap_end` (wir haben ihn im vorherigen [part](linux-bootstrap-2.md) berechnet) und gibt 1 zurück, falls genügend Speicher für `n` verfügbar ist.

Das ist alles. Jetzt haben wir eine einfache API für den Heap und können den Videomodus einrichten.

## Videomodus einrichten

Jetzt können wir direkt zur Initialisierung des Videomodus übergehen. Wir haben beim Aufruf von `RESET_HEAP()` in der Funktion `set_video` angehalten. Als nächstes folgt der Aufruf von `store_mode_params`, der die Videomodusparameter in der Struktur `boot_params.screen_info` speichert, die in der Header-Datei [include/uapi/linux/screen_info.h](https://github.com/torvalds/linux/blob/v4.16/include/uapi/linux/screen_info.h) definiert ist.

Wenn wir uns die Funktion `store_mode_params` ansehen, sehen wir, dass sie mit einem Aufruf der Funktion `store_cursor_position` beginnt. Wie Sie dem Funktionsnamen entnehmen können, werden Informationen über den Cursor abgerufen und gespeichert.

Zunächst initialisiert `store_cursor_position` zwei Variablen vom Typ `biosregs` mit `AH = 0x3` und ruft die BIOS-Unterbrechung `0x10` auf. Nachdem die Unterbrechung erfolgreich ausgeführt wurde, werden Zeile und Spalte in den Registern `DL` und `DH` zurückgegeben. Zeile und Spalte wird in dem Feld `orig_x` und `orig_y` der Struktur `boot_params.screen_info` gespeichert.

Nachdem `store_cursor_position` ausgeführt wurde, wird die Funktion `store_video_mode` aufgerufen. Es wird nur der aktuelle Videomodus abgerufen und in `boot_params.screen_info.orig_video_mode` gespeichert.

Danach überprüft `store_mode_params` den aktuellen Videomodus und setzt das `video_segment`. Nachdem das BIOS die Steuerung an den Bootsektor übertragen hat, gelten die folgenden Adressen für den Videospeicher:

```
0xB000:0x0000 	32 Kb 	Monochrome Text Video Memory
0xB800:0x0000 	32 Kb 	Color Text Video Memory
```

Daher setzen wir die Variable `video_segment` auf `0xb000`, wenn der aktuelle Videomodus MDA, HGC oder VGA im Schwarzweißmodus ist, oder auf `0xb800`, wenn der aktuelle Videomodus im Farbmodus ist. Nach dem Einstellen der Adresse des Videosegments muss die Schriftgröße in `boot_params.screen_info.orig_video_points` gespeichert werden:

```C
set_fs(0);
font_size = rdfs16(0x485);
boot_params.screen_info.orig_video_points = font_size;
```

Zuerst setzen wir 0 in das `FS`-Register über die Funktion `set_fs`. Funktionen wie `set_fs` haben wir bereits im vorigen Teil gesehen. Sie sind alle in [arch/x86/boot/boot.h](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/boot.h) definiert. Als nächstes lesen wir den Wert, der sich an der Adresse `0x485` befindet (dieser Speicherort wird zum Abrufen der Schriftgröße verwendet) und speichern die Schriftgröße in `boot_params.screen_info.orig_video_points`.

```C
x = rdfs16(0x44a);
y = (adapter == ADAPTER_CGA) ? 25 : rdfs8(0x484)+1;
```

Als nächstes erhalten wir die Anzahl der Spalten nach Adresse `0x44a` und die Anzahl der Zeilen nach Adresse `0x484` und speichern sie in `boot_params.screen_info.orig_video_cols` und `boot_params.screen_info.orig_video_lines`. Danach ist die Ausführung von `store_mode_params` beendet.

Als nächstes sehen wir die Funktion `save_screen`, die nur den Inhalt des Bildschirms auf dem Heap speichert. Diese Funktion sammelt alle Daten, die wir in den vorherigen Funktionen erhalten haben (wie die Zeilen und Spalten und so weiter) und speichert sie in der `saved_screen`-Struktur, die wie folgt definiert ist:

```C
static struct saved_screen {
	int x, y;
	int curx, cury;
	u16 *data;
} saved;
```

Anschließend wird geprüft, ob auf dem Heap freier Speicherplatz vorhanden ist.

```C
if (!heap_free(saved.x*saved.y*sizeof(u16)+512))
		return;
```

bei ausreichendem Platz wird Speicher im Heap reserviert und `saved_screen` darin speichert.

Der nächste Aufruf ist `probe_cards (0)` aus der Quellcode-Datei [arch/x86/boot/video-mode.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/video-mode.c). Es geht über alle video_cards und sammelt die Anzahl der von den Karten bereitgestellten Modi. Hier ist der interessante Teil, wir können die Schleife sehen:

```C
for (card = video_cards; card < video_cards_end; card++) {
  /* collecting number of modes here */
}
```

Aber `video_cards` wird nirgendwo deklariert. Die Antwort ist einfach: Jeder im x86-Kernel-Setup-Code dargestellte Videomodus hat eine Definition, die folgendermaßen aussieht:

```C
static __videocard video_vga = {
	.card_name	= "VGA",
	.probe		= vga_probe,
	.set_mode	= vga_set_mode,
};
```

wobei `__videocard` ein Makro ist:

```C
#define __videocard struct card_info __attribute__((used,section(".videocards")))
```

was bedeutet, dass die `card_info` Struktur:

```C
struct card_info {
	const char *card_name;
	int (*set_mode)(struct mode_info *mode);
	int (*probe)(void);
	struct mode_info *modes;
	int nmodes;
	int unsafe;
	u16 xmode_first;
	u16 xmode_n;
};
```

sich im `.videocards` Segment befindet. Schauen wir uns das Linker-Skript [arch/x86/boot/setup.ld](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld) an, dort finden wir folgendes:

```
	.videocards	: {
		video_cards = .;
		*(.videocards)
		video_cards_end = .;
	}
```

Das bedeutet, dass `video_cards` nur eine Speicheradresse ist und alle `card_info`-Strukturen in diesem Segment platziert sind. Das bedeutet, dass alle `card_info`-Strukturen zwischen`video_cards` und `video_cards_end` platziert sind, sodass wir eine Schleife verwenden können, um alle zu durchlaufen. Nach der Ausführung von `probe_cards` haben wir eine Reihe von Strukturen wie`static __videocard video_vga` mit den ausgefüllten `nmodes` (die Anzahl der Videomodi).

Nachdem die Funktion `probe_cards` ausgeführt wurde, wechseln wir in die Hauptschleife der Funktion`set_video`. Es gibt eine Endlosschleife, die versucht, den Videomodus mit der Funktion `set_mode` einzurichten oder ein Menü ausgibt, wenn `vid_mode=ask` an die Kernel-Befehlszeile übergeben wurde oder der Videomodus undefiniert ist.

Die Funktion `set_mode` ist in [video-mode.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/video-mode.c) definiert und erhält nur einen Parameter, `mode`, das ist die Anzahl der Videomodi (wir haben diesen Wert aus dem Menü oder zu Beginn von`setup_video` aus dem Kernel-Setup-Header erhalten).

Die Funktion `set_mode` überprüft den `mode` und ruft die Funktion `raw_set_mode` auf. Der `raw_set_mode` ruft die `set_mode`-Funktion der ausgewählten Karte auf, d. H. `card->set_mode(struct mode_info*)`. Zugriff auf diese Funktion erhalten wir über die Struktur `card_info`. Jeder Videomodus definiert diese Struktur mit Werten, die abhängig vom Videomodus gefüllt werden (zum Beispiel für `vga` ist es die Funktion `video_vga.set_mode`. Siehe das obige Beispiel der `card_info`-Struktur für `vga`). `video_vga.set_mode` ist`vga_set_mode`, der den vga-modus prüft und die entsprechende funktion aufruft:

```C
static int vga_set_mode(struct mode_info *mode)
{
	vga_set_basic_mode();

	force_x = mode->x;
	force_y = mode->y;

	switch (mode->mode) {
	case VIDEO_80x25:
		break;
	case VIDEO_8POINT:
		vga_set_8font();
		break;
	case VIDEO_80x43:
		vga_set_80x43();
		break;
	case VIDEO_80x28:
		vga_set_14font();
		break;
	case VIDEO_80x30:
		vga_set_80x30();
		break;
	case VIDEO_80x34:
		vga_set_80x34();
		break;
	case VIDEO_80x60:
		vga_set_80x60();
		break;
	}
	return 0;
}
```

Jede Funktion, die den Videomodus einstellt, ruft nur den BIOS-Interrupt `0x10` mit einem bestimmten Wert im Register `AH` auf.

Nachdem wir den Videomodus eingestellt haben, übergeben wir ihn an `boot_params.hdr.vid_mode`.

Als nächstes wird `vesa_store_edid` aufgerufen. Diese Funktion speichert einfach die Informationen zu [EDID](https://en.wikipedia.org/wiki/Extended_Display_Identification_Data) (**E**xtended **D**isplay **I**dentification **D**ata) für den Kernel. Danach wird `store_mode_params` erneut aufgerufen. Wenn `do_restore` eingestellt ist, wird der Bildschirm in einen früheren Zustand zurückversetzt.

Damit ist die Einrichtung des Videomodus abgeschlossen und wir können in den geschützten Modus wechseln.

## Letzte Vorbereitung vor dem Übergang in den geschützten Modus

Wir können den letzten Funktionsaufruf - `go_to_protected_mode` - in [arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c) sehen. Wie der Kommentar besagt: `Do the last things and invoke protected mode`. Sehen wir uns also die letzten Dinge an und wechseln Sie in den geschützten Modus.

Die Funktion `go_to_protected_mode` ist in [arch/x86/boot/pm.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/pm.c) definiert. Es enthält einige Funktionen, die die letzten Vorbereitungen treffen, bevor wir in den geschützten Modus springen können. Schauen wir uns das an und versuchen Sie zu verstehen, was es tut und wie es funktioniert.

Zuerst wird die Funktion `realmode_switch_hook` in `go_to_protected_mode` aufgerufen. Diese Funktion ruft den Haken des Real-Modus-Schalters auf, falls vorhanden, und deaktiviert [NMI](http://en.wikipedia.org/wiki/Non-maskable_interrupt). Hooks werden verwendet, wenn der Bootloader in einer feindlichen Umgebung ausgeführt wird. Weitere Informationen zu Hooks finden Sie im [Boot-Protokoll](https://www.kernel.org/doc/Documentation/x86/boot.txt) (siehe **ADVANCED BOOT LOADER HOOKS**).

Zunächst wird die Funktion `realmode_switch_hook` im `go_to_protected_mode` aufgerufen. Diese Funktion öffnet den Haken des Real-Modus-Schalters (falls vorhanden) und deaktiviert [NMI](http://en.wikipedia.org/wiki/Non-maskable_interrupt). Hooks werden verwendet, wenn der Bootloader in einer feindlichen Umgebung ausgeführt wird. Weitere Informationen zu Hooks finden Sie unter [Boot Log](https://www.kernel.org/doc/Documentation/x86/boot.txt) (siehe **ADVANCED BOOT LOADER HOOKS**).

Der `realmode_switch`-Hook präsentiert einen Zeiger auf die 16-Bit-Real-Mode-Far-Subroutine, die nicht maskierbare Interrupts deaktiviert. Nachdem der `realmode_switch`-Haken (für mich nicht vorhanden) überprüft wurde, ist die Funktion für nicht maskierbare Interrupts (NMI) deaktiviert:

```assembly
asm volatile("cli");
outb(0x80, 0x70);	/* Disable NMI */
io_delay();
```

Zuerst gibt es eine Inline-Assembly-Anweisung mit einem Befehl `cli`, der das Interrupt-Flag (`IF`) löscht. Danach werden externe Interrupts deaktiviert. Die nächste Zeile deaktiviert NMI (nicht maskierbarer Interrupt).

Ein Interrupt ist ein Signal an die CPU, das von Hardware oder Software ausgegeben wird. Nachdem ein solches Signal empfangen wurde, setzt die CPU die aktuelle Befehlssequenz aus, speichert ihren Zustand und übergibt die Steuerung an den Interrupt-Handler. Nachdem der Interrupt-Handler seine Arbeit beendet hat, gibt er die Kontrolle an den unterbrochenen Befehl zurück. Nicht maskierbare Interrupts (NMI) sind Interrupts, die unabhängig von der Berechtigung immer verarbeitet werden. Sie können nicht ignoriert werden und werden normalerweise verwendet, um auf nicht behebbare Hardwarefehler hinzuweisen. Wir werden uns jetzt nicht mit den Details von Interrupts befassen, aber wir werden sie in den kommenden Beiträgen diskutieren.

Kommen wir zurück zum Code. In der zweiten Zeile sehen wir, dass wir das Byte `0x80` (deaktiviertes Bit) in `0x70` (das CMOS-Adressregister) schreiben. Danach wird die Funktion `io_delay` aufgerufen. `io_delay` verursacht eine kleine Verzögerung und sieht so aus:

```C
static inline void io_delay(void)
{
	const u16 DELAY_PORT = 0x80;
	asm volatile("outb %%al,%0" : : "dN" (DELAY_PORT));
}
```

Um ein Byte an den Port `0x80` auszugeben, sollte die Verzögerung genau 1 Mikrosekunde betragen. Wir können also jeden beliebigen Wert (in unserem Fall den Wert von `AL`) in den `0x80`-Port schreiben. Nach dieser Verzögerung ist die Ausführung der Funktion `realmode_switch_hook` beendet und wir können zur nächsten Funktion übergehen.

Die nächste Funktion ist `enable_a20`, die die [A20-Zeile](http://en.wikipedia.org/wiki/A20_line) aktiviert. Diese Funktion ist in [arch/x86/boot/a20.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/a20.c) definiert und versucht das A20 Tor mit verschiedenen Methoden zu aktivieren. Die erste ist die Funktion `a20_test_short`, die prüft, ob A20 bereits mit der Funktion`a20_test` aktiviert ist oder nicht:

```C
static int a20_test(int loops)
{
	int ok = 0;
	int saved, ctr;

	set_fs(0x0000);
	set_gs(0xffff);

	saved = ctr = rdfs32(A20_TEST_ADDR);

        while (loops--) {
		wrfs32(++ctr, A20_TEST_ADDR);
		io_delay();	/* Serialize and make delay constant */
		ok = rdgs32(A20_TEST_ADDR+0x10) ^ ctr;
		if (ok)
			break;
	}

	wrfs32(saved, A20_TEST_ADDR);
	return ok;
}
```

Zuerst setzen wir `0x0000` in das `FS`-Register und `0xffff` in das `GS`-Register. Als nächstes lesen wir den Wert an der Adresse `A20_TEST_ADDR` (dieser ist `0x200`) und setzen diesen Wert in die Variablen `saved` und `ctr`.

Als nächstes schreiben wir einen aktualisierten `ctr`-Wert in `fs:A20_TEST_ADDR` oder `fs:0x200` mit der Funktion `wrfs32`, dann die Verzögerung auf 1 ms und lesen dann den Wert aus dem `GS`-Register in die Adresse `A20_TEST_ADDR+0x10`. In einem Fall, in dem die `a20`-Leitung deaktiviert ist, wird die Adresse überlappt, in einem anderen Fall, wenn sie nicht Null ist, ist die `a20`-Leitung bereits aktiviert.

Wenn A20 deaktiviert ist, versuchen wir es mit einer anderen Methode zu aktivieren, welche Sie in `a20.c` finden. Dies kann beispielsweise durch einen Aufruf des BIOS-Interrupts `0x15` mit `AH=0x2041` geschehen.

Wenn die Funktion `enable_a20` fehlgeschlagen ist, drucken Sie eine Fehlermeldung und rufen Sie die Funktion`die` auf. Sie können sich an die erste Quellcodedatei erinnern, in der wir folgendes gestartet haben - [arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S):

```assembly
die:
	hlt
	jmp	die
	.size	die, .-die
```

Nachdem das A20-Gate erfolgreich aktiviert wurde, wird die Funktion `reset_coprocessor` aufgerufen:

```C
outb(0, 0xf0);
outb(0, 0xf1);
```

Diese Funktion löscht den mathematischen Coprozessor, indem Sie `0` in `0xf0` schreiben und setzt ihn dann zurück, indem Sie `0` in `0xf1` schreiben.

Danach wird die Funktion `mask_all_interrupts` aufgerufen:

```C
outb(0xff, 0xa1);       /* Mask all interrupts on the secondary PIC */
outb(0xfb, 0x21);       /* Mask all but cascade on the primary PIC */
```

Dies maskiert alle Interrupts auf dem sekundären PIC (Programmable Interrupt Controller) und dem primären PIC mit Ausnahme von IRQ2 auf dem primären PIC.

Und nach all diesen Vorbereitungen können wir den tatsächlichen Übergang in den geschützten Modus sehen.

## Einrichtung der Interrupt Descriptor Table

Jetzt richten wir die Interrupt Descriptor Table (IDT) in der Funktion `setup_idt` ein:

```C
static void setup_idt(void)
{
	static const struct gdt_ptr null_idt = {0, 0};
	asm volatile("lidtl %0" : : "m" (null_idt));
}
```

Hiermit wird die Interrupt-Deskriptor-Tabelle eingerichtet (beschreibt Interrupt-Handler usw.). Im Moment ist der IDT nicht installiert (wir werden ihn später sehen), aber jetzt laden wir den IDT einfach mit der Anweisung `lidtl`. `null_idt` enthält die Adresse und Größe des IDT, aber im Moment sind sie nur Null. `null_idt` ist eine `gdt_ptr` Struktur, definiert als:

```C
struct gdt_ptr {
	u16 len;
	u32 ptr;
} __attribute__((packed));
```

wo wir die 16-Bit-Länge (`len`) des IDT und den 32-Bit-Zeiger darauf sehen können (Weitere Details über das IDT und Unterbrechungen werden in den nächsten Beiträgen zu sehen sein). `__attribute__((packed))` bedeutet, dass die Größe von `gdt_ptr` die minimal erforderliche Größe ist. Die Größe des `gdt_ptr` beträgt hier also 6 Bytes oder 48 Bits. (Als nächstes werden wir den Zeiger auf `gdt_ptr` in das`GDTR`-Register laden und Sie werden sich vielleicht aus dem vorherigen Beitrag daran erinnern, dass es eine Größe von 48 Bit hat).

## Einrichtung der Global Descriptor Table

Als nächstes folgt der Aufbau der Global Descriptor Table (GDT). Wir können die `setup_gdt`-Funktion sehen, die die GDT einrichtet (Sie können darüber im nachträglichen [Kernel-Boot-Vorgang. Teil 2.](linux-bootstrap-2.md#protected-mode) nachlesen.). In dieser Funktion gibt es eine Definition des `boot_gdt` Arrays, die die Definition der drei Segmente enthält:

```C
static const u64 boot_gdt[] __attribute__((aligned(16))) = {
	[GDT_ENTRY_BOOT_CS] = GDT_ENTRY(0xc09b, 0, 0xfffff),
	[GDT_ENTRY_BOOT_DS] = GDT_ENTRY(0xc093, 0, 0xfffff),
	[GDT_ENTRY_BOOT_TSS] = GDT_ENTRY(0x0089, 4096, 103),
};
```

für Code, Daten und TSS (Task State Segment). Wir werden das Task-Status-Segment vorerst nicht verwenden, es wurde hinzugefügt, um Intel VT glücklich zu machen, wie wir in der Kommentarzeile sehen können (wenn Sie interessiert sind, können Sie das Commit finden, das es beschreibt - [hier](https://github.com/torvalds/linux/commit/88089519f302f1296b4739be45699f06f728ec31)). Schauen wir uns `boot_gdt` an. Zuallererst ist zu beachten, dass es das Attribut `__attribute __ ((aligniert (16)))` hat. Dies bedeutet, dass diese Struktur um 16 Bytes ausgerichtet wird.

Schauen wir uns ein einfaches Beispiel an:

```C
#include <stdio.h>

struct aligned {
	int a;
}__attribute__((aligned(16)));

struct nonaligned {
	int b;
};

int main(void)
{
	struct aligned    a;
	struct nonaligned na;

	printf("Not aligned - %zu \n", sizeof(na));
	printf("Aligned - %zu \n", sizeof(a));

	return 0;
}
```

Technisch gesehen muss eine Struktur, die ein `int`-Feld enthält, 4 Byte groß sein, aber eine `aligned` Struktur benötigt 16 Byte, um im Speicher gespeichert zu werden:

```
$ gcc test.c -o test && test
Not aligned - 4
Aligned - 16
```

Der `GDT_ENTRY_BOOT_CS` hat hier den Index - 2,`GDT_ENTRY_BOOT_DS` ist `GDT_ENTRY_BOOT_CS 1` und so weiter. Er beginnt bei 2, weil der erste ein obligatorischer Null-Deskriptor ist (Index - 0) und der zweite nicht verwendet wird (Index - 1). .

`GDT_ENTRY` ist ein Makro, das Flags, Base, Limit und Builds eines GDT-Eintrags übernimmt. Betrachten wir zum Beispiel den Code-Segment-Eintrag. `GDT_ENTRY` nimmt die folgenden Werte an:

- base - 0
- limit - 0xfffff
- flags - 0xc09b

Was bedeutet das? Die Basisadresse des Segments ist 0 und das Limit (Größe des Segments) ist - `0xfffff` (1 MB). Schauen wir uns die Fahnen an. Es ist `0xc09b` und es wird:

```
1100 0000 1001 1011
```

in binär. Versuchen wir zu verstehen, was jedes Bit bedeutet. Wir werden alle Teile von links nach rechts durchgehen:

- 1 - (G) granularity bit
- 1 - (D) if 0 16-bit segment; 1 = 32-bit segment
- 0 - (L) executed in 64-bit mode if 1
- 0 - (AVL) available for use by system software
- 0000 - 4-bit length 19:16 bits in the descriptor
- 1 - (P) segment presence in memory
- 00 - (DPL) - privilege level, 0 is the highest privilege
- 1 - (S) code or data segment, not a system segment
- 101 - segment type execute/read/
- 1 - accessed bit

Sie können mehr über jedes Bit im vorherigen [Beitrag](linux-bootstrap-2.md) oder in den [Intel® 64- und IA-32-Architekturen Software-Entwicklerhandbüchern 3A](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html) nachlesen.

Danach erhalten wir die Länge der GDT mit:

```C
gdt.len = sizeof(boot_gdt)-1;
```

Wir erhalten die Größe von `boot_gdt` und subtrahieren 1 (die letzte gültige Adresse in der GDT).

Als nächstes erhalten wir einen Zeiger auf die GDT mit:

```C
gdt.ptr = (u32)&boot_gdt + (ds() << 4);
```

Hier bekommen wir die Adresse von `boot_gdt` und addieren sie zu der Adresse des Datensegments, welches um 4 Bits nach links verschoben wurde (denken Sie daran, dass wir uns jetzt im Real-Modus befinden).

Zuletzt führen wir den Befehl `lgdtl` aus, um die GDT in das GDTR-Register zu laden:

```C
asm volatile("lgdtl %0" : : "m" (gdt));
```

## Aktueller Übergang in den geschützten Modus

Dies ist das Ende der Funktion `go_to_protected_mode`. Wir haben den IDT und den GDT geladen, Interrupts deaktiviert und können nun die CPU in den geschützten Modus schalten. Der letzte Schritt ist das Aufrufen der Funktion `protected_mode_jump` mit zwei Parametern:

```C
protected_mode_jump(boot_params.hdr.code32_start, (u32)&boot_params + (ds() << 4));
```

welche in [arch/x86/boot/pmjump.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/pmjump.S) definiert ist.

Es werden zwei Parameter benötigt:

- Adresse des Einstiegspunkts für den geschützten Modus
- Adresse von `boot_params`

Schauen wir uns `protected_mode_jump` an. Wie ich oben geschrieben habe, finden Sie es in `arch/x86/boot/pmjump.S`. Der erste Parameter befindet sich im `eax`-Register und der zweite im `edx`-Register.

Zunächst geben wir die Adresse von `boot_params` in das esi-Register und die Adresse des Codesegmentregisters `cs` in `bx`. Danach verschieben wir `bx` um 4 Bits und fügen es dem Speicherplatz mit der Bezeichnung `2` hinzu (das ist `(cs << 4) + in_pm32`, die physikalische Adresse auf die gesprungen wird nach dem Übergang in den 32-Bit-Modus) und springen zu "1". Danach wird `in_pm32` im Label` 2` mit `(cs << 4) in_pm32` überschrieben.

Als nächstes setzen wir das Datensegment und das Task-Statussegment bei den Registern `cx` und` di` mit:

```assembly
movw	$__BOOT_DS, %cx
movw	$__BOOT_TSS, %di
```

Wie Sie oben lesen können, hat `GDT_ENTRY_BOOT_CS` den Index 2 und jeder GDT-Eintrag ist 8 Byte, also wird` CS` zu `2 * 8 = 16`, `__BOOT_DS` ist 24 usw.

Als nächstes setzen wir das `PE`-Bit (Protection Enable) im `CR0`-Steuerregister:

```assembly
movl	%cr0, %edx
orb	$X86_CR0_PE, %dl
movl	%edx, %cr0
```

und mache einen weiten Sprung in den geschützten Modus:

```assembly
	.byte	0x66, 0xea
2:	.long	in_pm32
	.word	__BOOT_CS
```

wobei:

- `0x66` ist das Präfix für die Operandengröße, mit dem 16-Bit- und 32-Bit-Code gemischt werden können
- `0xea` - ist der Sprung-Opcode
- `in_pm32` ist der Segmentoffset im Schutzmodus, dessen Wert `(cs << 4) + in_pm32` vom Realmodus abgeleitet ist
- `__BOOT_CS` ist das Codesegment, zu dem wir springen möchten.

Danach sind wir endlich im geschützten Modus:

```assembly
.code32
.section ".text32","ax"
```

Schauen wir uns die ersten Schritte im geschützten Modus an. Zunächst richten wir das Datensegment ein mit:

```assembly
movl	%ecx, %ds
movl	%ecx, %es
movl	%ecx, %fs
movl	%ecx, %gs
movl	%ecx, %ss
```

Wenn Sie aufgepasst haben, können Sie sich daran erinnern, dass wir `$__BOOT_DS` im `cx`-Register gespeichert haben. Jetzt füllen wir es mit allen Segmentregistern außer `cs` (`cs` ist bereits `__BOOT_CS`).

Richten Sie einen gültigen Stack für das Debuggen ein:

```assembly
addl	%ebx, %esp
```

Der letzte Schritt vor dem Sprung in den 32-Bit-Einstiegspunkt besteht darin, die Mehrzweckregister zu löschen:

```assembly
xorl	%ecx, %ecx
xorl	%edx, %edx
xorl	%ebx, %ebx
xorl	%ebp, %ebp
xorl	%edi, %edi
```

Und springe zum Ende des 32-Bit-Einstiegspunktes:

```
jmpl	*%eax
```

Denken Sie daran, dass `eax` die Adresse des 32-Bit-Eintrags enthält (wir haben ihn als ersten Parameter an `protected_mode_jump` übergeben).

Das ist alles. Wir sind im geschützten Modus und halten am Einstiegspunkt an. Im nächsten Teil werden wir sehen was als nächstes passiert.

## Fazit

Dies ist das Ende des dritten Teils über die Internas des Linux-Kernels. Im nächsten Teil werden wir uns die ersten Schritte ansehen, die wir im geschützten Modus unternehmen und in den [long mode](http://en.wikipedia.org/wiki/Long_mode) übergehen.

Wenn Sie Fragen oder Anregungen haben, schreiben Sie mir einen Kommentar oder senden Sie einen Ping an [twitter](https://twitter.com/0xAX).

**Bitte beachten Sie, dass Englisch nicht meine Muttersprache ist und ich mich für etwaige Unannehmlichkeiten wirklich entschuldige. Wenn Sie Fehler finden, senden Sie mir bitte PR an [linux-insides](https://github.com/0xAX/linux-internals).**

## Links

- [VGA](http://en.wikipedia.org/wiki/Video_Graphics_Array)
- [VESA BIOS Extensions](http://en.wikipedia.org/wiki/VESA_BIOS_Extensions)
- [Data structure alignment](http://en.wikipedia.org/wiki/Data_structure_alignment)
- [Non-maskable interrupt](http://en.wikipedia.org/wiki/Non-maskable_interrupt)
- [A20](http://en.wikipedia.org/wiki/A20_line)
- [GCC designated inits](https://gcc.gnu.org/onlinedocs/gcc-4.1.2/gcc/Designated-Inits.html)
- [GCC type attributes](https://gcc.gnu.org/onlinedocs/gcc/Type-Attributes.html)
- [Previous part](linux-bootstrap-2.md)
