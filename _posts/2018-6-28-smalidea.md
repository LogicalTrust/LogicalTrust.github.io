---
layout: post
title: "[PL] Smalidea - debugowanie aplikacji Android bez kodu źródłowego"
published: false
date: 2018-07-05
author: Michal
---

## Wstęp
Posiadając aplikację tylko w formie pliku APK, na przykład po zainstalowaniu jej z Google Play Store, niemożliwe jest debugowanie jej w standardowy sposób. Wynika to z braku dostępu do kodu źródłowego. APK zawiera jedynie bajtkod dla maszyny wirtualnej Dalvik i ART, przechowywany w plikach o rozszerzeniu dex. 

Z pomocą przychodzi smalidea. Jest to wtyczka dla IntelliJ IDEA i Android Studio, która dodaje obsługę języka smali (czyli asemblera bajtkodu Dalvika), a w tym również możliwość debugowania.

### Wymagania:
- IntelliJ IDEA lub Android Studio
- Android SDK
- urządzenie z Androidem lub emulator
- JDK 8 -  [http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)
- apktool - [https://ibotpeaches.github.io/Apktool/install/](https://ibotpeaches.github.io/Apktool/install/)
- smalidea-0.05.zip lub nowszy - [https://bitbucket.org/JesusFreke/smali/downloads/](https://bitbucket.org/JesusFreke/smali/downloads/)

Proces instalacji i uruchomienia będzie opisywany na przykładzie Kali Linuksa oraz Android Studio 3.1, które domyślnie zainstalowało Android SDK w katalogu domowym. Debugowaniu zostanie poddana aplikacja DIVA Android.

## Przygotowanie projektu

### Dezasemblacja

Najpierw powinniśmy zacząć od wydobycia APK z urządzenia. Do tego można wykorzystać narzędzie adb należące do Android SDK.

{% highlight x %}
$ adb pull /data/app/jakhar.aseem.diva-1/base.apk diva.apk
{% endhighlight %}

A do przeprowadzenia dezasemblacji skorzystamy z apktool. Rozpakuje on wskazany przez nas plik APK, dezasemblując przy tym pliki dex za pomocą wbudowanego narzędzia baksmali.

{% highlight x %}
$ apktool d -o DivaSmali diva.apk
{% endhighlight %}


### Dodanie projektu

Będąc w posiadaniu kodu smali, możemy go teraz dodać do Android Studio. W tym celu należy zaimportować projekt, wybierając opcję z istniejącymi źródłami.

Zrzuty prezentują sposób na dodanie projektu przy pierwszym uruchomieniu Android Studio. Za to z kolejnymi uruchomieniami będziemy mogli dodawać projekty wybierając File -> New -> Import Project...

[![Import project](/images/smalidea/import.png)](/images/smalidea/import.png){: .center-image }

[![Create project from existing sources](/images/smalidea/import-existing-sources.png)](/images/smalidea/import-existing-sources.png){: .center-image }


### Instalacja smalidea

Ten krok wykonujemy tylko dla pierwszego dodawanego projektu. Po instalacji, wtyczka będzie dostępna również dla nowych projektów.

Otwieramy ustawienia Android Studio wybierając File -> Settings. Odnajdujemy sekcję Plugins i instalujemy wtyczkę z dysku (Install plugin from disk...).

[![Install plugin from disk...](/images/smalidea/install-plugin.png)](/images/smalidea/install-plugin.png){: .center-image }

Po zakończeniu restartujemy Android Studio.


### Konfiguracja projektu

Samo dodanie projektu nie jest wystarczające. Musimy jeszcze odpowiednio go przygotować. Domyślnym widokiem w projekcie jest Android, zmieniamy go jednak na widok plików projektu.

[![Project Files](/images/smalidea/project-files.png)](/images/smalidea/project-files.png){: .center-image }

Dzięki temu zobaczymy katalog z kodem smali (jakhar/aseem/diva), który oznaczymy jako katalog główny.

[![Mark directory as -> Sources Root](/images/smalidea/sources-root.png)](/images/smalidea/sources-root.png){: .center-image }

Teraz pozostaje już tylko ustawienie odpowiedniego SDK dla projektu. Otwieramy ustawienia projektu (File -> Project Structure) i odnajdujemy sekcję Project. W niej jako Project SDK wybieramy Android API, które powinniśmy mieć wcześniej zainstalowane.

[![File -> Project Structure -> Project -> Project SDK -> Android API](/images/smalidea/project-sdk.png)](/images/smalidea/project-sdk.png){: .center-image }

## Łączenie z debuggerem
### Przygotowanie debuggera

Zaczynamy od uruchomienia narzędzia Android Device Monitor. Po włączeniu go, powinniśmy zobaczyć nasze urządzenie wraz z listą możliwych do debugowania procesów i przypisanymi do nich portami. W tym przypadku proces jakhar.aseem.diva jest dostępny na porcie 8600.

{% highlight x %}
$ Android/Sdk/tools/monitor
{% endhighlight %}
[![Android Device Manager](/images/smalidea/adm.png)](/images/smalidea/adm.png){: .center-image }

Wracamy do Android Studio, w którym musimy dodać konfigurację pozwalającą na zdalne uruchomienie projektu. Łączyć będziemy się z lokalnym hostem, na porcie powiązanym z naszą docelową aplikacją.

[![Run -> Edit Configuration](/images/smalidea/edit-configuration.png)](/images/smalidea/edit-configuration.png){: .center-image }
[![Add -> Remote](/images/smalidea/add-configuration-remote.png)](/images/smalidea/add-configuration-remote.png){: .center-image }
[![Host: localhost, Port: 8600](/images/smalidea/configuration-localhost-8600.png)](/images/smalidea/configuration-localhost-8600.png){: .center-image }

#### Wskazówka na przyszłość

Numery portów są przydzielane procesom dynamicznie, więc przy kolejnej próbie debugowania może on ulec zmianie. Rozwiązaniem jest ustawienie portu 8700 (zamiast pokazanego wyżej 8600). Jest on statyczny i zawsze odnosi się do aktualnie zaznaczonego procesu na liście w Android Device Monitor.

### Uruchomienie debuggera

Po przygotowaniu konfiguracji, możemy ją wykorzystać do rozpoczęcia procesu debugowania.

[![Run -> Debug](/images/smalidea/run-debug.png)](/images/smalidea/run-debug.png){: .center-image }

O pomyślnym nawiązaniu połączenia poinformuje nas komunikat w konsoli Android Studio.
{% highlight x %}
Connected to the target VM, address: 'localhost:8600', transport: 'socket'
{% endhighlight %}


## Proces debugowania

Po udanym połączeniu z debuggerem możemy w wybranych miejscach w kodzie ustawiać breakpointy. A po zatrzymaniu się, wykonywać instrukcje krok po kroku oraz korzystać z wyrażeń do podglądania zawartości zmiennych lokalnych.

W ramach przykładu sprawdzimy dwa miejsca w aplikacji. 

### Przykład 1

Zadanie Access Control Part 3 wymaga wprowadzenia poprawnego numeru PIN, aby uzyskać dostęp do prywatnych notatek. Aplikacja porównuje wartość wprowadzoną przez użytkownika z numerem wcześniej zdefiniowanym w aplikacji. Za pomocą debuggera możemy zatrzymać się w miejscu, w którym odczytywany jest wcześniej zdefiniowany PIN i podejrzeć go.

[![Run -> Debug](/images/smalidea/pin-breakpoint.png)](/images/smalidea/pin-breakpoint.png){: .center-image }

W ten sposób dowiadujemy się, że zdefiniowany PIN to 9087.

### Przykład 2

W zadaniu Input Validation Issues 1 występuje błąd typu SQL Injection. Zapytanie tworzone jest za pomocą konkatenacji stringów. Zatrzymując się na wywołaniu zapytania na bazie i korzystając z funkcji Evaluate expression, możemy się dokładnie przyjrzeć jak zostało ono zbudowane.

[![Run -> Debug](/images/smalidea/sql-breakpoint.png)](/images/smalidea/sql-breakpoint.png){: .center-image }
[![Run -> Debug](/images/smalidea/expression.png)](/images/smalidea/expression.png){: .center-image }

Znalezienie odpowiedniego miejsca do ustawienia breakpointa może być trudne. Dlatego możemy sobie pomóc korzystając z dekompilatora 
(np. [jadx](https://github.com/skylot/jadx)). Wgląd w zdekompilowany kod może ułatwić namierzenie charakterystycznych miejsc w plikach smali.

[![Run -> Debug](/images/smalidea/jadx-sql.png)](/images/smalidea/jadx-sql.png){: .center-image }


## Aplikacje z wyłączoną możliwością debugowania

W przypadku większości aplikacji w wersjach produkcyjnych, proces nie będzie widoczny w Android Device Monitor. Wynikać to będzie z faktu, że aplikacja nie została zbudowana z włączoną opcją debugowania. W takiej sytuacji konieczne będzie przepakowanie aplikacji, odpowiednio modyfikując jej manifest.

Mając już na komputerze plik APK, należy go rozpakować narzędziem apktool. Uruchamiając apktool tym razem, można skorzystać z opcji *s*. Sprawi ona, że odkodowane zostaną zasoby aplikacji, w tym AndroidManifest.xml, pozostawiając pliki .dex nienaruszone, co skróci czas wykonywania operacji.

{% highlight x %}
$ apktool d -s base.apk
{% endhighlight %}


Po rozpakowaniu edytujemy plik AndroidManifest.xml. W elemencie *application* dodajemy atrybut *debuggable* ustawiony na *true*.

[![Run -> Debug](/images/smalidea/debuggable.png)](/images/smalidea/debuggable.png){: .center-image }

Proces budowania nowego APK, razem z podpisywaniem, opisany jest w [poprzednim wpisie](/backdoor-drozer) w części *Zbudowanie nowego APK*.