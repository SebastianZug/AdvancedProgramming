<!--

author:   Sebastian Zug & Georg Jäger
email:    sebastian.zug@informatik.tu-freiberg.de & Georg.Jaeger@informatik.tu-freiberg.de
version:  0.0.1
language: de
narrator: Deutsch Female

import: https://raw.githubusercontent.com/LiaTemplates/Rextester/master/README.md

-->

# Vorlesung Advanced Programming

In dieser und der nächsten Folge wollen wir Ihnen einen Überblick zum Robot Operating System (ROS) geben und dabei insbesondere den Unterschied zwischen ROS 1 und ROS 2 herausarbeiten.

Eine interaktive Version des Kurses finden Sie unter [Link](https://liascript.github.io/course/?https://raw.githubusercontent.com/SebastianZug/SoftwareprojektRobotik/master/02_SpeicherUndPointer.md#1)

**Zielstellung der heutigen Veranstaltung**

+ Unterscheidung von Stack und Heap
+ Gegenüberstellung von Basis-Pointern und Smart-Pointern
+ Herausforderungen beim Speichermanagement
+ Templates unter C++

--------------------------------------------------------------------------------

## Komponenten des Speichers

Wie wird der Speicher von einem C++ Programm eigentlich verwaltet? Wie wird diese Struktur ausgehend vom Start eines Programmes aufgebaut?

<!--
style="width: 70%; max-width: 860px; display: block; margin-left: auto; margin-right: auto;"
-->
```ascii
        ------>  +----------------------------+
   höhere        | Kommandozeilen Parameter   |    (1)
   Adresse       | und Umgebungsvariablen     |
                 +----------------------------+
                 | Stack                      |    (2)
                 |............................|
                 |             |              |
                 |             v              |
                 |                            |
                 |             ^              |
                 |             |              |
                 |............................|
                 | Heap                       |    (3)
                 +----------------------------+
                 | uninitialisierte Daten     |    (4)
                 +----------------------------+
                 | initialisierte Daten       |    (5)
  kleinere       +----------------------------+
  Adresse        | Programmcode               |    (6)
       ------>   +----------------------------+
```

|     | Speicherbestandteil    | engl. | Bedeutung                              |
| --- | ---------------------- | ----- | -------------------------------------- |
| 1   | Parameter              |       |                                        |
| 2   | Stack                  |       | LIFO Datenstruktur                     |
| 3   | Heap                   |       | "Haldenspeicher"                       |
| 4   | Uninitialisierte Daten | .bss  |                                        |
| 5   | Initialisierte Daten   | .data | Globale und statische Daten,           |
| 6   | Programmcode           | .text | teilbar, häufig als read only Speicher |


### Stack vs Heap
<!--
  comment: StackvsHeap.cpp
  ..............................................................................
  1. Illustriere die Bedeutung ind der Unterscheidung von Heap und Stack! Verweis auf die Warnings
  ```cpp
  int* CreateArray(){
     int array[] = {1,2,3,4,5};
     return array;
   }
  int main()
  {
     int* a = CreateArray();
     std::cout << a [2] << std::endl;
  ...
  ```
  ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
-->

**Zunächst mal ganz praktisch, was passiert auf dem Stack?**

Ausgangspunkt unserer Untersuchung ist ein kleines Programm, das mit dem gnu Debugger `gdb` analysiert wurde:

```
g++ -m32 StackExample.cpp -o stackExample
gdb stackExample
(gdb) disas main
(gdb) disas calc
```

```cpp                     StackExample.cpp
#include <iostream>

int calc(int factor1, int factor2){
  return factor1 * factor2;
}

int main()
{
  int num1 {0x11};
  int num2 {0x22};
  int result {0};
  result = calc(num1, num2);
  std::cout << result << std::endl;
  return EXIT_SUCCESS;
}
```
@Rextester.CPP

Was passiert beim starten des Programmes und beim  Aufruf der Funktion `calc` "unter der Haube"? Schauen wir zunächst die Einrichtung des Stacks von Seiten der `main` funktion bis zur Zeile 12.

```
0x06ed <+10>:    push   %ebp
0x06ee <+11>:    mov    %esp,%ebp
0x06f0 <+13>:    push   %ebx
0x06f1 <+14>:    push   %ecx
0x06f2 <+15>:    sub    $0x10,%esp
... # Initialisierung der Variablen
0x0700 <+29>:    movl   $0x11,-0x14(%ebp)
0x0707 <+36>:    movl   $0x22,-0x10(%ebp)
0x070e <+43>:    movl   $0x0,-0xc(%ebp)
... # Aufruf des Unterprogramms
0x071b <+56>:    call   0x6cd <calc(int, int)>
0x0720 <+61>:    add    $0x8,%esp
0x0723 <+64>:    mov    %eax,-0xc(%ebp)
0x0726 <+67>:    sub    $0x8,%esp
... # Aufruf Betriebssystemschnittstell für Ausgabe
0x0733 <+80>:    call   0x570 <_ZNSolsEi@plt>
....
```

<!--
style="width: 100%; max-width: 860px; display: block; margin-left: auto; margin-right: auto;"
-->
```ascii
                                    kleinere Adresse

            +----------------> |                           |
            |                  +---------------------------+
            |          rbp-x1c | ????                      | ^
            |                  +---------------------------+ |
            |          rbp-x14 | num1                      | |  4Byte
      +------------+           +---------------------------+ |  = 0x10
rsb   |stackpointer|   rbp-x10 | num2                      | |
      +------------+           +---------------------------+ |
                       rbp-x0c | result                    | v
                               +---------------------------+
                       rbp-x08 | ecx                       |  Gesicherte
                               +---------------------------+  Register
                       rbp-x04 | ebx                       |
      +------------+           +---------------------------+
rbp   |basepointer |------->   | Alter Framepointer        |  neuer Frame
      +------------+           +---------------------------+  ..........
                               | Alter Instruktionspointer |
                               | ...                       |

                                     größere Adresse
```

Nun rufen wir die Funktion `calc` auf und führen die Berechnung aus. Dafür
wird ein neuer Stackframe angelegt. Wie entwickelt sich der Stack ausgehend von
dem zugehörigen Assemblercode weiter?

```
0x06cd <+0>:  push   %ebp
0x06ce <+1>:  mov    %esp,%ebp
0x06da <+13>: mov    0x8(%ebp),%eax
0x06dd <+16>: imul   0xc(%ebp),%eax
0x06e1 <+20>: pop    %ebp
0x06e2 <+21>: ret
```

<!--
style="width: 100%; max-width: 860px; display: block; margin-left: auto; margin-right: auto;"
-->
```ascii
                                    kleinere Adresse

            +----------------> | ...                       |
            |                  +---------------------------+
            |           +----> |   Alter Framepointer   ...|.......
            |           |      +---------------------------+      .
            |           |      | Alter Instruktionspointer |      .
            |           |      +---------------------------+      .
            |           |      | ????                      |      .
            |           |      +---------------------------+      .
            |           |      | num1                      |      .
      +------------+    |      +---------------------------+      .
rsb   |stackpointer|    |      | num2                      |      .
      +------------+    |      +---------------------------+      .
                        |      | result                    |      .
                        |      +---------------------------+      .
                        |      | ecx                       |      .
                        |      +---------------------------+      .
                        |      | ebx                       |      .
      +------------+    |      +---------------------------+      .
rbp   |basepointer |----+      | Alter Framepointer        | <.....
      +------------+           +---------------------------+
                               | Alter Instruktionspointer |
                               +---------------------------+
                               | ...                       |

                                     größere Adresse
```

**Heap**

Der Heap ist ein dedizierter Teil des RAM, in dem von der Applikation Speicher dynamisch belegt werden kann. Speichergrößen werden explizit angefordert und wieder frei gegeben.

Unter C(!) erfolgt dies mit den Funktionen der Standardbibliothek. C++ übernimmt diese Funktionalität.

```c
#include <stdlib.h>
void *malloc(size_t size);
void *calloc(size_t n, size_t size);
void *realloc(void *ptr, size_t size)
free(void *ptr);
```

C++ erhöht den Komfort für den Entwickler und implementiert ein alternatives Konzept.

```cpp
new «Datentyp» »(«Konstruktorargumente»)«
delete «Speicheradresse»
```

|                   | `new`                        | `malloc`                        |
| ----------------- | ---------------------------- | ------------------------------- |
| Bedeutung         | Schlüsselwort, Operator      | Funktion der Standardbibliothek |
| Rückgabewert      | Pointer vom Typ des Objektes | `void` Pointer                  |
| Fehlerfall        | Ausnahme                     | NULL Pointer                    |
| Größe             | wird vom Compiler bestimmt   | muss manuell festgelegt werden  |
| Überschreibarkeit | ja                           |                                 |


**Beispiel**

```
#include <iostream>

struct Bruch{
  int zaehler = 1;
  int nenner = 1;

  Bruch(int z, int n) : zaehler{z}, nenner{n} {};
  void print(){
    std::cout << zaehler << "/" << nenner << std::endl;
  }
};

int main(){
  Bruch einhalb {1,2};
  Bruch* einviertel = static_cast<Bruch*>(malloc(sizeof(Bruch)));
  einviertel->zaehler = 1;
  einviertel->nenner = 4;
  einviertel->print();
  Bruch* einachtel = new Bruch(1,8);
  einachtel->print();
  return EXIT_SUCCESS;
}
```

[Link auf pythontutor](http://pythontutor.com/cpp.html#code=%23include%20%3Ciostream%3E%0A%0Astruct%20Bruch%7B%0A%20%20int%20zaehler%20%3D%201%3B%0A%20%20int%20nenner%20%3D%201%3B%0A%0A%20%20Bruch%28int%20z,%20int%20n%29%20%3A%20zaehler%7Bz%7D,%20nenner%7Bn%7D%20%7B%7D%3B%0A%20%20void%20print%28%29%7B%0A%20%20%20%20std%3A%3Acout%20%3C%3C%20zaehler%20%3C%3C%20%22/%22%20%3C%3C%20nenner%20%3C%3C%20std%3A%3Aendl%3B%0A%20%20%7D%0A%7D%3B%0A%0Aint%20main%28%29%7B%0A%20%20Bruch%20einhalb%20%7B1,2%7D%3B%0A%20%20Bruch*%20einviertel%20%3D%20static_cast%3CBruch*%3E%28malloc%28sizeof%28Bruch%29%29%29%3B%0A%20%20einviertel-%3Ezaehler%20%3D%201%3B%0A%20%20einviertel-%3Enenner%20%3D%204%3B%0A%20%20einviertel-%3Eprint%28%29%3B%0A%20%20Bruch*%20einachtel%20%3D%20new%20Bruch%281,8%29%3B%0A%20%20einachtel-%3Eprint%28%29%3B%0A%20%20return%20EXIT_SUCCESS%3B%0A%7D&curInstr=17&mode=display&origin=opt-frontend.js&py=cpp&rawInputLstJSON=%5B%5D)


**Zusammenfassung**

| Parameter                   | Stack                          | Heap                            |
| --------------------------- | ------------------------------ | ------------------------------- |
| Allokation und Deallokation | Automatisch durch den Compiler | Manuel durch den Programmierer  |
| Kosten                      | gering                         | ggf. höher durch Fragmentierung |
| Flexibilität                | feste Größe                    | Anpassungen möglich             |


**Und noch mal am Beispiel**

```cpp                     StackvsHeap.cpp
#include <iostream>

class MyClass{
  public:
    MyClass()
    {
      std::cout << "Constructor executed" << std::endl;
    }
    ~MyClass()
    {
      std::cout << "Deconstructor executed" << std::endl;
    }
};

int main()
{
  {  // Scope I
     MyClass A;
  }
  {  // Scope II
     MyClass* B = new MyClass;
  }
  return EXIT_SUCCESS;
}
```
@Rextester.CPP


## (Raw-)Pointer / Referenzen

**Was war noch mal eine Referenz?**

C und C\+\+ unterstützen das Konzept des **Zeigers**, die sich von den meisten anderen Programmiersprachen unterscheiden. Andere Sprachen wie C#, C++(!), Java, Python etc. implementieren **Referenzen**.

Oberflächlich betrachtet sind sich Referenzen und Zeiger sehr ähnlich, in beiden
Fällen greifen wir indirekt auf einen Speicher zu:

+ Ein Zeiger ist eine Variable, die die Speicheradresse einer anderen Variablen enthält. Ein Zeiger muss mit dem Operator \* dereferenziert werden, um auf den Speicherort zuzugreifen. Pointer können auch ins "nichts" zeigen. C++11 definiert dafür den `nullptr`.

+ Eine Referenz ist ein Alias, das heißt ein anderer Name für eine bereits vorhandene Variable. Eine Referenz wird wie ein Zeiger implementiert, indem die Adresse eines Objekts gespeichert wird. Entsprechend kann eine Referenz auch nur für ein bestehendes Objekt angelegt werden.

Die Funktionsweise soll an folgendem Beispiel verdeutlicht werden:

```cpp                     Constructor.cpp
#include <iostream>

void DoSomeCalculationsByValue(int number){
  number = number * 2;
}

void DoSomeCalculationsByPointer(int* number){
  *number = *number * 2;
}

void DoSomeCalculationsByReference(int& number){
  number = number * 2;
}

int main()
{
  int number {1};
  DoSomeCalculationsByValue(number);
  std::cout << number << std::endl;
  int* ptr = &number;
  DoSomeCalculationsByPointer(ptr);
  std::cout << *ptr << std::endl;
  int& ref = number;
  DoSomeCalculationsByReference(ref);
  std::cout << ref << std::endl;
  return EXIT_SUCCESS;
}
```
@Rextester.CPP

**Nachteile bei der Verwendung von Raw Pointern**

> "Raw pointers have been a pain in the backside for most students learning C++ in the last decades. They had a multi-purpose role as raw memory iterators, nullable, changeable nullable references and devices to manage memory that no-one really owns. This lead to a host of bugs and vulnerabilities and headaches and decades of human life spent debugging, and the loss of joy in programming completely." [Link](https://arne-mertz.de/2018/04/raw-pointers-are-gone/)

> "In modernem C++ werden Rohzeiger nur in kleinen Codeblöcken mit begrenztem Gültigkeitsbereich, in Schleifen oder Hilfsfunktionen verwendet, in denen Leistung ausschlaggebend ist und keine Verwirrung über den Besitzer entstehen kann." [Microsoft](https://docs.microsoft.com/de-de/cpp/cpp/smart-pointers-modern-cpp?view=vs-2019)

## Smart Pointer
<!--
  comment: SelfDesignedUniquePointer.cpp
  ..............................................................................
  1.
     ```cpp
     class myPointer{
       private:
         MyClass* m_Ptr;
       public:
         myPointer(MyClass* ptr) : m_Ptr (ptr) {}
         ~myPointer(){
           delete m_Ptr;
         }
     };
     int main()
     {
       {
          //myPointer C(new MyClass());
          myPointer C = new MyClass();
       }
     }
     ```
   ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
-->

Das folgende Beispiel greift einen Nachteil der Raw Pointer nochmals auf, um das
Konzept höherabstrakter Zeigerformate herzuleiten. Wie können wir sicherstellen,
dass im folgenden das Objekt auf dem Heap zerstört wird, wenn wir den Scope verlassen?

```cpp                     SelfDesignedUniquePointer.cpp
#include <iostream>

class MyClass{
  public:
    MyClass()
    {
      std::cout << "Constructor executed" << std::endl;
    }
    ~MyClass()
    {
      std::cout << "Deconstructor executed" << std::endl;
    }
};

int main()
{
  {
     MyClass* A = new MyClass();
     // ... Some fancy things happen here ...
     delete A;
  }
  return EXIT_SUCCESS;
}
```
@Rextester.CPP

Wir kombinieren eine stackbasierten Pointerklasse, die die den Zugriff auf das
eigentliche Datenobjekt kapselt. Mit dem verlassen des Scopes wird auch der
allozierte Speicher freigegeben.

```cpp
class myPointer{
  private:
    MyClass* m_Ptr;
  public:
    myPointer(MyClass* ptr) : m_Ptr (ptr) {}
    ~myPointer(){
      delete m_Ptr;
    }
};

//myPointer C(new MyClass()); // oder
myPointer C = new MyClass();
```

Die Wirkungsweise eines intelligenten C++-Zeigers ähnelt dem Vorgehen bei der Objekterstellung in Sprachen wie C#: Sie erstellen das Objekt und überlassen es dann dem System, das Objekt zur richtigen Zeit zu löschen. Der Unterschied besteht darin, dass im Hintergrund keine separate Speicherbereinigung ausgeführt wird – der Arbeitsspeicher wird durch die C++-Standardregeln für den Gültigkeitsbereich verwaltet, sodass die Laufzeitumgebung schneller und effizienter ist.

C++11 implementiert verschiedene Pointer-Klassen für unterschiedliche Zwecke. Diese werden im folgenden vorgestellt:

+ std::unique_ptr — smart pointer der den Zugriff auf eine dynamisch allokierte Ressource verwaltet.
+ std::shared_ptr — smart pointer der den Zugriff auf eine dynamisch allokierte Ressource verwaltet, wobei mehrere Instanzen für ein und die selbe Ressource bestehen können.
+ std::weak_ptr — analog zum `std::shared_ptr` aber ohne Überwachung der entsprechenden Pointerinstanzen.

### unique Pointers
<!--
  comment: UniquePointer.cpp
  ..............................................................................
  1. Diskutiere den Sinn von
     ```cpp
      std::make_unique<MyClass>();  // exception save
      //std::unique_ptr<MyClass> A (new MyClass());
      //  1. Step - Generation of MyClass Instance
      //  2. Step - Generation of unique_ptr Instance
     ```
     Auf beiden Ebenen kann ein Fehler passieren, entweder generieren wir einen
     Pointer der ins Nichts zeigt oder eine Memory Leak.
  2. Versuch einer Kopieroperation
     ```cpp
     std:unique_ptr<MyClass> B = A;
     ```
    ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
-->

Die `unique` Pointer stellen sicher, dass das eigentliche Objekt nur durch einen
einzelnen Pointer adressiert wird. Wird der entsprechende Scope verlassen, wird das Konstrukt automatisch gelöscht.

```cpp                     UniquePointer.cpp
#include <iostream>
#include <memory>   //<-- Notwendiger Header

class MyClass{
  public:
    MyClass(){
      std::cout << "Constructor executed" << std::endl;
    }
    ~MyClass(){
      std::cout << "Deconstructor executed" << std::endl;
    }
    void print(){
      std::cout << "That's all!" << std::endl;
    }
};

int main()
{
  {
     std::unique_ptr<MyClass> A (new MyClass());
     //std::unique_ptr<MyClass> B = new MyClass();
     A->print();
  }
  return EXIT_SUCCESS;
}
```
@Rextester.CPP

> Merke: Zu jedem Zeitpunkt verweist nur eine Instanz des `unique_ptr` auf eine Ressource. Die Idee lässt keine Kopien zu.

Wie wird das softwaretechnisch abgefangen? Die entsprechenden Methoden wurden mit "delete" markiert und generieren entsprechend eine Fehlermeldung.

```cpp
unique_ptr(const _Myt&) = delete;
_Myt& operator=(const _Myt&) = delete;
```

Dies ist dann insbesondere bei der Übergabe von `unique_ptr` an Funktionen von Bedeutung.

```cpp    UniquePointerHandling.cpp
#include <iostream>
#include <memory>

void callByValue(std::unique_ptr<std::string> input){
    std::cout << *input << std::endl;
}

void callByReference(const std::unique_ptr<std::string> & input){
    std::cout << *input << std::endl;
}

void callByRawPointer(std::string* input){
    std::cout << *input << std::endl;
}

int main()
{
  // Variante 1
  //std::unique_ptr<std::string> A (new std::string("Hello World"))
  // Variante 2
  std::unique_ptr<std::string> A = std::make_unique<std::string>("Hello World");
  // Ohne Veränderung der Ownership
  callbyValue(A);                     // Compilerfehler!!!
  callByReference(A);
  callByRawPointer(A.get());
  // Mit Übergabe der Ownership
  callByValue(std::move(A));
  //std::cout << A << std::endl;
  return EXIT_SUCCESS;
}
```
@Rextester.CPP

Diese konzeptionelle Einschränkung bringt aber einen entscheidenden Vorteil mit sich. Der `unique` Pointer wird allein durch eine Adresse repräsentiert. Es ist kein weiterer Overhead für die Verwaltung des Konstrukts notwendig!
Vergleichen Sie dazu die Darstellung unter [cppreference](https://de.cppreference.com/w/cpp/memory/unique_ptr)


### sharded Pointers
<!--
  comment: SharedPointer.cpp
  ..............................................................................
  1. Füge einen neuen Scope ein, in dem verschiedene shared_ptr auf ein Objekt
     zeigen.
     ```cpp
     int main()
     {
        std::shared_ptr<MyClass> A;
       {
          std::shared_ptr<MyClass> B = make_shared<MyClass>();
          A = B;
       }
       A->print();
       return EXIT_SUCCESS;
     }
     ```
  2. Präsentiere die API des Shared POinter
     ```cpp
     B.use_count();
     B.unique();
     ```
  ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
-->

std::shared_ptr sind intelligente Zeiger, die ein Objekt über einen Zeiger "verwalten". Mehrere shared_ptr Instanzen können das selbe Objekt besitzen. Das Objekt wird zerstört, wenn:

+ der letzte shared_ptr, der das Objekt besitzt, zerstört wird oder
+ dem letzten shared_ptr, der das Objekt besitzt, ein neues Objekt mittles operator= oder reset() zugewiesen wird.

Das Objekt wird entweder mittels einer delete-expression oder einem benutzerdefiniertem deleter zerstört, der dem shared_ptr während der Erzeugung übergeben wird.

```cpp                     SharedPointer.cpp
#include <iostream>
#include <memory>   //<-- Notwendiger Header

class MyClass{
  public:
    MyClass(){
      std::cout << "Constructor executed" << std::endl;
    }
    ~MyClass(){
      std::cout << "Deconstructor executed" << std::endl;
    }
    void print(){
      std::cout << "That's all!" << std::endl;
    }
};

int main()
{
  {
     std::shared_ptr<MyClass> B = std::make_shared<MyClass>();
     B->print();
  }
  std::cout << "Scope left" << std::endl;
  return EXIT_SUCCESS;
}
```@Rextester.CPP

Gegenwärtig sind Arrays noch etwas knifflig in der Representation und Handhabung. C++17 schafft hier erstmals die Möglichkeit einer adquaten Handhabung (vgl. StackOverflow Diskussionen).

### weak Pointers

Ein weak_ptr ist im Grunde ein shared Pointer, der die Referenzanzahl nicht erhöht. Es ist definiert als ein intelligenter Zeiger, der eine nicht-besitzende Referenz oder eine schwache Referenz auf ein Objekt enthält, das von einem anderen shared Pointer verwaltet wird.

```cpp                     WeakPointer.cpp
#include <iostream>
#include <memory>

std::weak_ptr<int> gw;

void f()
{
    if (auto spt = gw.lock()) { // Has to be copied into a shared_ptr before usage
	  std::cout << *spt << "\n";
    }
    else {
        std::cout << "gw is expired\n";
    }
}

int main()
{
    {
      auto sp = std::make_shared<int>(42);
      gw = sp;
    	f();
    }
    f();
}
```@Rextester.CPP

### Regeln für den Einsatz von Pointern

Eine schöne Übersicht zu den Fehlern im Zusammenhang mit Smart Pointer ist unter
[https://www.acodersjourney.com](https://www.acodersjourney.com/top-10-dumb-mistakes-avoid-c-11-smart-pointers/)
zu finden.

## Arten von Templates

Templates (englisch für Schablonen oder Vorlagen) sind ein Mittel zur Typparametrierung in C++. Templates ermöglichen generische Programmierung und **typsichere** Container.
In der C++-Standardbibliothek werden Templates zur Bereitstellung typsicherer Container, wie z. B. Listen, und zur Implementierung von generischen Algorithmen, wie z.B. Sortierverfahren, verwendet.

### Funktionstemplates
<!--
  comment: FunctionOverloading.cs
  ..............................................................................
  Einführen einer neuen Methode `print(double value)` und Delegation von `print(int value)` über einen expliziten Cast. Ergänzung einer Methode für `const char[]`
  ```cpp
  void print(const char* value){
    std::cout << value;
  }
  void print (double value){
    std::cout << value << std::endl;
  }
  void print (int value){
    print ((double)value);
  }
  ```
  ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
-->
Welche Möglichkeiten haben wir unter C++ schon kennengelernt, die einenn variablen
Umgange von identischen Funktionsaufrufen mit unterschiedlichen Typen realsieren?
Dabei unterstützen uns Cast-Operatoren und das Überladen von Funktionen.

```cpp                     FunctionOverloading.cpp
#include <iostream>

void print (int value){
  std::cout << value << std::endl;
}

int main()
{
  print(5);
  print(10.234);
  //print("TU Freiberg");
  return EXIT_SUCCESS;
}
```
@Rextester.CPP

Ein Funktions-Template (auch fälschlich Template-Funktion genannt) verhält sich wie eine Funktion, die Argumente verschiedener Typen akzeptiert oder unterschiedliche Rückgabetypen liefert. Die C++-Standardbibliothek enthält eine Vielzahl von Funktionstemplates, die folgendem Muster einer selbstdefinierten Funktion entsprechen.

```cpp                     FunctionTemplate.cpp
#include <iostream>

template<typename T>          // Definition des Typalias T
void print (T value){         // Innerhalb von print wirkt T als Platzhalter
  std::cout << value << std::endl;
}

int main()
{
  print(5);
  print(10.234);
  print("TU Freiberg");
  return EXIT_SUCCESS;
}
```
@Rextester.CPP

Was macht der Compiler daraus? Lassen Sie uns prüfen, welche Symbole erzeugt wurden. Offenbar wurde aus unserem Funktionstemplate entsprechend der Referenzierung 3 unterschiedliche Varianten generiert. Mit `c++filt` kann der Klarname rekonstruiert werden.

```bash
>g++ Templates.cpp
>nm a.out | grep print
0000000000000a26 W _Z5printIdEvT_
00000000000009f2 W _Z5printIiEvT_
0000000000000a64 W _Z5printIPKcEvT
>c++filt _Z5printIdEvT_
void print<double>(double)
```

Damit sollten folgende Begriffe festgehalten werden:

+ ein Funktionstemplate definiert ein Muster oder eine Schablone für die eigentliche Funktionalität
+ das Funktionstemplate wird initialisiert, in dem die Templateparameter festgelegt werden.
+ der Compiler übprüft ob die angegebenen Datentypen zu einer validen Funktionsdefinition führen und generiert diese bei einem positiven Match.
+ der Compiler versucht solange alle Funktionstemplates mit den angegebenen Datentypen zu initialisieren, bis diese erfolgreich war, oder kein weiteres Funktionstemplate zur Verfügung steht.


Warum funktioniert das Konzept bisher so gut? Alle bisher verwendeten Typen bedienen den Operator `<<` für die Stream-Ausgabe. Eine Datenstruktur, die dieses Kriterium nicht erfüllt würde einen Compilerfehler generieren.

Zwei Fragen bleiben noch offen:

| Frage                                                                            | Antwort                                                                                                                                                                                                                                                                                                                                                                              |
|:---------------------------------------------------------------------------------|:-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Muss der Templateparameter zwingend angegeben werden?                            | Nein, wenn Sie im nachfolgenden Codebeispiel die Funktion `print(int value)` entfernen, funktioniert die Codegenerierung noch immer. Der Compiler erkennt den Typen anhand des übergebenen Wertes.                                                                                                                                                                                   |
| Ist ein Nebeneinander von Funktionstemplates und allgemeinen Funktionen möglich? | Ja, denn der Compiler versucht immer die *spezifischste* Funktion zu nutzen. Das heißt, zunächst werden alle nicht-templatisierten Funktionen in betracht gezogen. Im zweiten Schritt werden teilweise-spezialisierte Funktionstemplates herangezogen und erst zu letzt werden vollständig templatisierte Funktionen genutzt. Untersuchen Sie das Beispiel mit `nm` und `c++filter`! |

```cpp                     FunctionTemplate.cpp
#include <iostream>

void print (int value){
  std::cout << "Die Funktion wird aufgerufen" << std::endl;
  std::cout << value << std::endl;
}

template<typename T>
void print (T value){
  std::cout << "Die Templatefunktion wird aufgerufen" << std::endl;
  std::cout << value << std::endl;
}

template<>
void print<float> (float value){
  std::cout << "Die spezialisierte Templatefunktion wird aufgerufen" << std::endl;
  std::cout << value << std::endl;
}

int main()
{
  print(5);

  double v = 5.0;
  print<float>(v);
  print(v);
  return EXIT_SUCCESS;
}
```
@Rextester.CPP

Insbesondere teilweise und vollständige Templatespezialisierungen ermöglichen es Ausnahmen von generellen Abbildungsregeln darzustellen.

```cpp                     TemplateSpecialization.cpp
#include <iostream>

template<typename T>
void print (T value){
  std::cout << value << std::endl;
}

template <> void print<bool> (bool value){
  std::cout << (value ? "true" : "false") << std::endl;
}

int main()
{
  print(5);
  print<bool>(true);
  return EXIT_SUCCESS;
}
```
@Rextester.CPP

Damit lassen sich dann insbesondere für Templates mit mehr als einem Typparameter komplexe Regelsets aufstellen:

```cpp
template<class T, class U>  // Generische Funktion
void f(T a, U b) {}

template<class T>           // Teilweise spezialisiertes Funktions-Template
void f(T a, int b) {}

template<>                  // Vollständig spezialisiert; immer noch Template
void f(int a, int b) {}
```

Daraus ergibt sich folgende Aufrufstruktur:

| Aufruf                | Adressierte Funktion                          |
|:----------------------|:----------------------------------------------|
| `f<int, int> (3,7);`  | Generische Funktion                           |
| `f<double> (3.5, 5);` | Überladenes Funktionstemplate                 |
| `f(3.5, 5);`          | Überladenes Funktionstemplate                 |
| `f(3, 5);`            | Vollständig spezialisiertes Funktionstemplate |

Methoden-Templates sind auch nur Funktionstemplates, dass heißt die gereade vorstellten Mechanismen lassen sich im Kontext Funktion, die einer Klasse zugeordnet ist analog umsetzen.

### Klassentemplates
<!--
  comment: ClassTemplate.cpp
  ..............................................................................
  1. Wie können wir unser Konstrukt auf dem Heap ablegen?
  ```cpp
  #include <memory>
  std::shared_ptr<OptionalVariable<double>> Para3 = std::make_shared<OptionalVariable<double>>(242.23424);
  std::cout << Para3->getVariable();
  ```
  2. Im Weiteren lassen sich auch für Member der Klasse Spezialisierungen definieren
  ```cpp
  template <>
  void OptionalVariable<std::string>::clear(){
     std::cout << "Lösche den String" << std::endl;
     variable = "-";
  }
  ```
  ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
-->

    {{0-1}}
*******************************************************************************

**Grundsätzliche Idee**

Ein Klassen-Template geht einen Schritt weiter und wendet die Template-Konzepte einheitlich auf eine Klasse an. Dies können selbstgeschriebene
Klassen sein oder Klassen, die zum Beispiel in der C++ Standardbibliothek
(STL) eingebettet sind und Container für Daten definieren.

Das folgende Beispiel definiert ein eigenes Klassentemplate, dass die Verwaltung einer Variablen übernimmt. Sofern ein gültiger Wert hinterlegt wurde, ist die entsprechende
Kontrollvariable gesetzt:

```cpp                    ClassTemplate.cpp
#include <iostream>
#include <list>
#include <string>

template <typename T>
class OptionalVariable{
     T variable;
     bool valid;
   public:
     OptionalVariable() : valid(false) {}
     OptionalVariable(T variable): variable(variable), valid(true) {}
     T getVariable(){
       if (!valid){
          std::cerr << "Variable not valid!" << std::endl;
       }
       return variable;
     }
     void clear(){
         valid = false;
     }
};

int main(){
    OptionalVariable<int> Para1 = OptionalVariable<int>(5);
    std::cout << Para1.getVariable();

    OptionalVariable<std::string> Para2 = OptionalVariable<std::string> ("Das ist ein Test");
    Para2.clear();
    std::cout << Para2.getVariable();
}
```
@Rextester.CPP

Auch in diesem Kontext kann eine Spezialisierung von Templates für bestimmte
Typen erfolgen. Spezialisieren Sie beispielsweise das Template für den Typ `std::string`
und setzen Sie in der Methode `clear` die zugehörige Variable auf einen leeren Ausdruck `""`.

Nur am Rande ... Was wären alternative C++ Umsetzungen für die genannte Anwendung:

+ Verwendung eines templatisierten SmartPointers ohne das eigene Template
+ Addressierung der Variablen mittels Zeiger und dessen Setzen auf `nullptr` für den Fall eines ungültigen Eintrages

*******************************************************************************

     {{1-2}}
*******************************************************************************

**Klassentemplates der STL**

Anwendungsseitig spielen Templates im Zusammenhang mit den Containern der STL
eine große Bedeutung. Die Datenstrukturen sind (analog zu C#) als Klassentemplates
realisiert.

```cpp      GenericListExample.cpp
#include <iostream>
#include <list>
#include <string>

class MyClass{
   private:
     std::string name;
   public:
     MyClass(std::string name) : name(name) {}
     std::string getName() const {return this->name;}
};

int main(){
    // Beispiel 1
    std::list< int > array = { 3, 5, 7, 11 };
    for(auto i = std::begin(array);
        i != std::end(array);
        ++i
    ){
        std::cout << *i << ", ";
    }

    // Beispiel 2
    std::list<MyClass> objects;

    objects.emplace_back("Hello World!");
    for (auto it = objects.begin(); it!=objects.end(); it++) {
      std::cout << it->getName() << std::endl;
    }
}
```
@Rextester.CPP


*******************************************************************************

     {{2-3}}
*******************************************************************************

**Mehrfache Template-Parameter**

Darüber hinaus können (wie auch bei den Funktionstemplates) mehrere Datentypen
in Klassentemplates angegeben werden. Der vorliegende Code illustriert dies
am Beispiel einer Klassifikation, die zum Beispiel mit einem neuronalen Netz
erfolgte. Ein Set von Features wird auf eine Klasse abgebildet.

```cpp                           Container.cpp
#include <iostream>
#include <list>
#include <string>
#include <iterator>

template<typename T, typename U>
class Classification{
   private:
     std::list<T> input;
     U category;
   public:
     template<typename ContainerType>
     Classification(ContainerType t, U u) : input(t.begin(), t.end()), category(u) {}
     void print(std::ostream& os) const{
        os << "Category: " << this->category << std::endl;
        for(const auto& v : this->input)
        {
            os << v << std::endl;
        }
     }
};

int main()
{
    std::list<int> samples {23, 2, 19, -12};
    std::string category  {"Person"};
    Classification<int, std::string> datasample {samples, category};
    datasample.print(std::cout);
}
```
@Rextester.CPP


*******************************************************************************

     {{3-4}}
*******************************************************************************


**Templates und Vererbung**

1. Vererbung von einem Klassentemplate auf ein Klassentemplate

Zwischen Klassentemplates können Vererbungsrelationen bestehen, wie zwischen konkreten Klassen. Dabei sind verschiedene Konfigurationen möglich:

+ die erbende Klasse nutzt den gleichen formalen Datentypen wie die Basisklasse (vgl. nachfolgendes Beispiel 1)
+ die erbende Klasse erweitert das Set der Templates um zusätzliche Parameter
+ die erbende Klasse konkretisiert einen oder alle Parameter
+ die Vererbungsrelation besteht zu einem formalen Datentyp, der dann aber eine Klasse sein muss. (Beispiel 2)

```cpp                 example.cpp
#include <iostream>
#include <list>
#include <string>

template<typename T>
class Base{
   private:
     T data;
   public:
     void set (const T& value){
        data = value;
     }
};

template<typename U>
class Derived: public Base<U>{
  public:
    void set (const U& value){
      Base<U>::set(value);
      // some additional operations happen here
    }
};

int main(){
    Derived<int> A;
    A.set(5);
}
```

Im Beispiel 2 erbt die abgeleitete Klasse unmittelbar vom Templateparameter.

```cpp                 InheritanceFromFormelType.cpp
#include <iostream>
#include <list>
#include <string>
#include <chrono>
#include <iomanip>
#include <sstream> // stringstream

class Data{
  private:
    double value;
  public:
    Data(double value): value(value) {}
    void set(double value) {
      value = value;
    }
    double get(){
      return value;
    }
};

template<typename T>
class TimeStampedData: public T{
  private:
    std::time_t timestamp;
  public:
    TimeStampedData(double value): T(value) {
      auto time_point = std::chrono::system_clock::now();
      timestamp = std::chrono::system_clock::to_time_t(time_point);
    }
    void print(std::ostream& os) const{
      std::stringstream ss;
      os << std::put_time(std::localtime(&timestamp), "%Y-%m-%d %X");
      os << " Value " << this.get();
    }
};

int main(){
    TimeStampedData<Data> A {5};
    A.print(std::cout);
}
```

Was ist kritisch an dieser Implementierung?

+ Die formelle Festlegung auf `double` in der Klasse `Data` schränkt die Wiederverwendbarkeit drastisch ein!
+ Das Klassentemplate setzt ein entsprechendes Interface voraus, dass einen Konstruktor, eine set-Funktion und ein Member data vom Typ double erwartet.

Wir müssen also prüfen, ob die Member des Templateparameters mit diesen Signaturen übereinstimmen.

```
#include <type_traits>

template<typename T>
class YourClass {

    YourClass() {
        // Compile-time check
        static_assert(std::is_base_of<BaseClass, T>::value, "type parameter of this class must derive from BaseClass");

        // ...
    }
}
```

Aktuell besteht keine Unterstützung von *constrains* analog zum Generics-Konzepts unter C#. Hier werden in dem akutellen C++20 Standards mit *concepts* neue Möglichkeiten definiert. Prüfen Sie ggf. ob Ihr Compiler diese unterstützt!

Zum Beispiel für den g++ unter ... https://gcc.gnu.org/projects/cxx-status.html

*******************************************************************************

## Template Parameter

     {{0-1}}
*******************************************************************************

Bei der Definition eines Templates kann entweder `class` oder aber `typename`
verwendet werden. Beide Formate sind in den meisten Fällen austauschbar.

```cpp
template<class T>
class Foo
{
};

template<typename T>
class Foo
{
};
```

Allerdings gibt es Unterschiede bei der Verwendung in Bezug auf Template-Templates (bis C++17) und bei abhängigen Typen.

>Beispiel !

*******************************************************************************

     {{1-2}}
*******************************************************************************

**1. Datentypen**

... waren Gegenstand des vorangegangenen Kapitels.

**2. Nichttyp-Parameter**

Nichttyp-Parameter sind Konstanten, mit denen Größen, Verfahren oder Prädikate als Template-Parameter übergeben werden können. Als Nichttyp-Template-Parameter sind erlaubt:

+ ganzzahlige Konstanten (inklusive Zeichenkonstanten),
+ Zeigerkonstanten (Daten- und Funktionszeiger, inklusive Zeiger auf Member-Variablen und -Funktionen) und
+ Zeichenkettenkonstanten

Verwendung finden Nichttyp-Parametern z. B. als Größenangabe bei std::array oder als Sortier- und Suchkriterium bei vielen Algorithmen der Standardbibliothek.

```cpp                    ConstantsAsTemplateParameters.cpp
#include <iostream>
#include <cstddef>          // Für den Typ size_t

template <std::size_t N, typename T>
void array_init(T (&array)[N], T const &startwert){
    for(std::size_t i=0; i<N; ++i)
        array[i]=startwert + i;
}

int main(){
    const std::size_t size {10};
    int A[size];
    array_init<size, int>(A, 6);
    for (unsigned int i=0; i < size; i++){
    	std::cout << A[i] << " ";
    }
    std::cout << std::endl;
    return EXIT_SUCCESS;
}
```
@Rextester.CPP

**Template-Templates**

Als Template-Parameter können aber auch wiederum Templates genutzt werden.

```cpp
template <template <typename, typename> class Container,
          typename Type>
class Example {
    Container<Type, std::allocator <Type> > myContainer;
    //...
};

// Anwendung
Example <std::deque, int> example;
```

Anwendung findet dies zum Beispiel in der Implementierung der `std::stack`
Klasse, die ohne weitere Parameter eine `std::deque` als Datenstruktur
verwendet. Diese wird in der Klassendeklaration als Standardparameter übergeben.
Der zugrunde liegende Container kann eine der Standard-Container-Klassenvorlagen
sein, der aber folgenden Vorgänge unterstützen muss:

+ empty
+ size
+ back
+ push_back
+ pop_back

```
template<
    class T,
    class Container = std::deque<T>    // <- Warum wird hier auf die erneute
>                                      // Templatisierung verzichtet?
class stack;
```

Für andere Container gibt es ähnliche Realisierungen. Der Beitrag von Herb Sutter gibt dazu eine fundierte Diskussion der Performanceunterschiede von `std::list`, `std::vector` und `std::deque` [Link](http://www.gotw.ca/gotw/054.htm).

## Aufgaben der Woche

1. Spielen Sie weitere Beispiele mit Smart-Pointern durch, die die spezifischen Anwendungsmöglichkeiten ausnutzen.
2. Erarbeiten Sie sich einen Überblick über die Verwendung der generischen Container unter C++.
