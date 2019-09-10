# Kernel-Boot-Prozess

In diesem Kapitel wird der Startvorgang des Linux-Kernels beschrieben. Hier sehen Sie eine Reihe von Beiträgen, die den gesamten Zyklus des Ladevorgangs des Kernels beschreiben:

- [Vom Bootloader zum Kernel](linux-bootstrap-1.md) - Beschreibt alle Schritte vom Einschalten des Computers bis zur Ausführung der ersten Anweisung des Kernels.
- [Erste Schritte im Kernel-Setup-Code](linux-bootstrap-2.md) - beschreibt die ersten Schritte im Kernel-Setup-Code. Sie sehen Heap-Initialisierung, Abfrage verschiedener Parameter wie EDD, IST und etc ...
- [Initialisierung des Videomodus und Übergang in den geschützten Modus](linux-bootstrap-3.md) - Beschreibt die Initialisierung des Videomodus im Kernel-Setup-Code und den Übergang in den geschützten Modus.
- [Übergang in den 64-Bit-Modus](linux-bootstrap-4.md) - Beschreibt die Vorbereitung für den Übergang in den 64-Bit-Modus und Einzelheiten zum Übergang.
- [Kernel-Dekomprimierung](linux-bootstrap-5.md) - beschreibt die Vorbereitung vor der Kernel-Dekomprimierung und Einzelheiten zur direkten Dekomprimierung.
- [Randomisierung der Kernel-Zufallsadresse](linux-bootstrap-6.md) - Beschreibt die Randomisierung der Linux-Kernel-Ladeadresse.

Dieses Kapitel stimmt mit `Linux-Kernel v4.17` überein.
