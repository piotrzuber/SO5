<div class="container">

## Zadanie 5

Najprawdopodobniej każdej i każdemu z nas zdarzyło się choć raz usunąć przez przypadek jakiś ważny plik. Problem ten wydaje się być powszechny, skoro wiele systemów operacyjnych i menedżerów plików wprowadza różnorodne zabezpieczenia (pytania typu „Czy na pewno chcesz usunąć ten plik?”, kosz systemowy, …) mające zapobiegać takim sytuacjom.

Celem tego zadania jest rozbudowanie serwera `mfs`, obsługującego system plików MINIX (MINIX File System), o prototypowe mechanizmy utrudniające przypadkowe usunięcie pliku w systemie MINIX.

W poniższym opisie słowo _plik_ jest używane w znaczeniu zwykłego pliku, nie katalogu.

### Usuwanie plików

Zmodyfikowany serwer działa tak samo jak normalny `mfs` za wyjątkiem operacji usuwania. Jeśli użytkownik będzie próbował usunąć plik (nie katalog), to działanie serwera zależy od aktywnego trybu:

#### Tryb A

W tym trybie próba usunięcia pliku powinna się po prostu nie powieść. Operacja usuwania kończy się błędem informującym, że takie działanie nie jest dozwolone (`EPERM`). Przykład:

<pre># ls
A.mode plik
# rm ./plik
rm: ./plik: Operation not permitted
# echo $?
1
# ls
A.mode plik
</pre>

Polecenie `echo $?` wyświetla status zakończenia (ang. _exit status_) poprzedniego polecenia.

#### Tryb B

Gdy aktywny jest ten tryb, aby skutecznie usunąć plik należy wywołać operację usuwania dwukrotnie. Pierwsza próba usunięcia kończy się błędem informującym, że operacja jest w toku (`EINPROGRESS`), a plik jest nadal dostępny. Można go odczytywać, edytować, a nawet odmontować i zamontować w innej maszynie dysk, na którym jest on zapisany. Dopiero ponowna próba usunięcia tego pliku rzeczywiście usuwa go z dysku. Jeśli użytkownik zorientuje się, że to nie ten plik chce usunąć, po pierwszym wywołaniu może wykonać operację zastąpienia tego pliku samym sobą (`mv ./plik ./plik`). Wtedy serwer „zapomina” o poprzedniej próbie usunięcia pliku i aby go teraz usunąć, należy znów wywołać operację usuwania dwukrotnie. Przykład:

<pre># ls
B.mode plik
# rm ./plik
rm: ./plik: Operation now in progress
# echo $?
1
# ls
B.mode plik
# mv ./plik ./plik
# rm ./plik
rm: ./plik: Operation now in progress
# ls
B.mode plik
# rm ./plik
# echo $?
0
# ls
B.mode
</pre>

#### Tryb C

Działając w tym trybie, serwer zamiast usunąć plik, zmienia mu nazwę, dodając do aktualnej sufiks `.bak`. Zatem, z punktu widzenia użytkownika, plik znika, ale powstaje jego kopia zapasowa. Usuwanie pliku będącego już kopią zapasową (pliku o nazwie kończącej się `.bak`) działa standardowo (tzn. bez tworzenia kopii zapasowej kopii zapasowej). Jeśli zmiana nazwy nie jest możliwa (np. po dodaniu sufiksu `.bak` nowa nazwa pliku przekroczy maksymalną dozwoloną długość), cała operacja usuwania powinna zakończyć się błędem informującym o przyczynie niepowodzenia, a plik nie powinien zostać usunięty. Przykład:

<pre># ls
C.mode plik
# rm ./plik
# echo $?
0
# ls
C.mode plik.bak
# rm ./plik.bak
# echo $?
0
# ls
C.mode
</pre>

### Sterowanie trybami

Aby serwer wykonywał usuwanie plików w sposób niestandardowy, użytkownik musi stworzyć plik o nazwie `A.mode`, `B.mode` lub `C.mode`, w zależności od tego, z którego trybu chce korzystać. Interfejs ten aktywuje wybrany tryb tylko dla usuwania plików leżących w tym samym katalogu co plik `{A,B,C}.mode`. Zatem w podkatalogu lub nadkatalogu użytkownik może korzystać z innego bądź żadnego trybu.

Aby dezaktywować wybrany tryb, należy usunąć plik `{A,B,C}.mode`. Usuwanie plików `{A,B,C}.mode` powinno działać zawsze w standardowy sposób, niezależnie od aktywnego trybu. Nie podlegają one specjalnemu usuwaniu! Jeśli użytkownik stworzy w danym miejscu więcej niż jeden plik `{A,B,C}.mode`, serwer może realizować operację usuwania w dowolny sposób, ale nie może ona uszkodzić systemu plików lub samego serwera.

### Wymagania i niewymagania

1.  Poprzez usuwanie pliku rozumiemy operację, jaką wykonuje serwer obsługujący system plików podczas użycia polecenia `rm` w powłoce lub jego odpowiednika w jakimś języku programowania.
2.  Usuwanie innych obiektów niż zwykłe pliki (np. katalogów) ma działać jak w niezmodyfikowanym serwerze, niezależnie od aktywnego trybu.
3.  Gdy żaden tryb specjalny nie jest aktywny (nie ma w danym katalogu żadnego z plików `{A,B,C}.mode`), usuwanie powinno działać jak w niezmodyfikowanym serwerze.
4.  Wszystkie pozostałe operacje i działania na plikach inne niż usuwanie powinny działać jak w niezmodyfikowanym serwerze.
5.  Modyfikacje serwera nie mogą powodować błędów w systemie plików: ma być on zawsze poprawny i spójny.
6.  Dyski przygotowane i używane przez niezmodyfikowany serwer `mfs` powinny być poprawnymi dyskami także dla zmodyfikowanego serwera. Nie wymagamy natomiast odwrotnej kompatybilności, tzn. dyski używane poprzez zmodyfikowany serwer nie muszą działać poprawnie z niezmodyfikowanym serwerem.
7.  Modyfikacje mogą dotyczyć tylko serwera `mfs` (czyli mogą dotyczyć tylko plików w katalogu `/usr/src/minix/fs/mfs`).
8.  Podczas działania zmodyfikowany serwer nie może wypisywać żadnych dodatkowych informacji na konsolę ani do rejestru zdarzeń (ang. _log_).
9.  Można założyć, że w testowanych przypadkach użytkownik będzie miał wystarczające uprawnienia do wykonania usuwania i wszystkich innych operacji, które potrzebuje wykonać zmodyfikowane usuwanie.
10.  Można założyć, że w testowanych przypadkach w systemie plików będą tylko zwykłe pliki (nie łącza, nie pseudo-urządzenia itp.) i katalogi.
11.  Można założyć, że na MINIX-e będzie zawsze ustawiona prawidłowa data i godzina, a rozwiązanie nie musi działać poprawnie przed 2019 i po 2037 roku.

### Wskazówki

1.  Aby skompilować i zainstalować zmodyfikowany serwer `mfs`, należy wykonać `make; make install` w katalogu `/usr/src/minix/fs/mfs`. Takimi poleceniami będzie budowane i instalowane oddanie rozwiązanie.
2.  Każde zamontowane położenie (ich listę wyświetli polecenie `mount`) obsługiwane jest przez nową instancję serwera `mfs`. Położenia zamontowane przed instalacją nowego serwera będą obsługiwane nadal przez jego starą wersję, więc, aby przetestować na nich zmodyfikowany serwer, należy je odmontować i zamontować ponownie lub zrestartować system.
3.  Aby zmodyfikowany serwer obsługiwał też korzeń systemu plików `/` należy wykonać dodatkowe kroki, ale radzimy nie testować na nim (i nie wymagamy tego) zmodyfikowanego usuwania.
4.  Do MINIX-a uruchomionego pod `qemu` można dołączać dodatkowe dyski twarde (i na nich testować swoje modyfikacje). Aby z tego skorzystać, należy:
    1.  Na komputerze-gospodarzu stworzyć plik będący nowym dyskiem, np.: `qemu-img create -f raw extra.img 1M`.
    2.  Podłączyć ten dysk do maszyny wirtualnej, dodając do parametrów, z jakimi uruchamiane jest `qemu`, parametry `-drive file=extra.img,format=raw,index=1,media=disk`, gdzie `index` to numer kolejny dysku (0 to główny dysk – główny obraz naszej maszyny).
    3.  Za pierwszym razem stworzyć na nowym dysku system plików mfs: `/sbin/mkfs.mfs /dev/c0d1`, gdzie `/dev/c0d<numer kolejny dodanego dysku>`.
    4.  Stworzyć pusty katalog (np. `mkdir /root/nowy`) i zamontować do niego podłączony dysk `mount /dev/c0d1 /root/nowy`.
    5.  Wszystkie operacje wewnątrz tego katalogu będą realizowane na zamontowanym w tym położeniu dysku.
    6.  Aby odmontować dysk, należy użyć polecenia `umount /root/nowy`.
5.  Szukając miejsca do przechowywania na dysku dodatkowych informacji o zapisanych na nim plikach można skorzystać z wymagania poprawnego działania systemu plików tylko od 2019 do 2037 roku.

### Rozwiązanie

Poniżej przyjmujemy, że ab123456 oznacza identyfikator studenta rozwiązującego zadanie. Należy przygotować łatkę (ang. _patch_) ze zmianami. Plik o nazwie `ab123456.patch` uzyskujemy za pomocą polecenia `diff -rupN`, tak jak w zadaniu 3\. Łatka będzie aplikowana przez umieszczenie jej w katalogu `/` nowej kopii MINIX-a i wykonanie polecenia `patch -p1 < ab123456.patch`. Należy zadbać, aby łatka zawierała tylko niezbędne różnice. W repozytorium, w katalogu `studenci/ab123456/zadanie5` należy umieścić tylko patch ze zmianami. Ostateczny termin umieszczenia rozwiązania w repozytorium to **<del>4 czerwca 2019, godz. 20.00</del> 9 czerwca 2019, godz. 23:59**.

### Ocenianie

Oceniana będą zarówno poprawność, jak i styl rozwiązania. Podstawą do oceny rozwiązania będą testy automatyczne i przejrzenie kodu przez sprawdzającego. Za poprawną i w dobrym stylu implementację obsługi usuwania plików w trybach A, B i C można otrzymać odpowiednio 1, 2 i 2 pkt. Rozwiązanie, w którym łatka nie nakłada się poprawnie, które nie kompiluje się lub powoduje `kernel panic` podczas uruchamiania otrzyma 0 pkt.
