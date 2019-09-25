

# API. Java.


---


Hva finnes av

# <u>bra</u> API-er for Java?


Note:
- noen som har noen biblioteker eller noe med APIer de liker spesielt godt?
- forslag:
	- Mockito?
	- Java Time API


---


<!-- .slide: data-state="ux-for-devs" -->

# UX for utviklere


Note:
- idiomatisk
  - konstruksjoner man kjenner igjen skal gjøre det man forventer
  - ikke finne på nye bruksområder for eksisterende "vokabular"
  - principle of least surprise
- ledes til å gjøre ting riktig
	- skal helst ikke se fornuftig ut dersom man gjør noe feil
- enhetlig
  - har man lært seg et konsept i API-et, skal man kjenne igjen konseptet andre steder


---


# Hva får man her?

```java
var duration = Duration.ofSeconds(2);
System.out.println(duration.getNano());
```

note:
Får '0'. Man har vært streng på idiomet at en "getter" er for å eksponere state. Det er _ikke_ snakk om å konvertere en varighet/tidslengde til en annen representasjon, altså 2 sekunder regnet om til nanosekunder.

----

## 🤮

```java
var calendar = Calendar.getInstance();
var date = calendar.getTime();
```

note:
Har en "kalender" (som egentlig er et punkt på tidslinjen), ber om å få "tiden", får kalenderen representert som en "dato" (som er en annen representasjon for et punkt på tidslinjen). Altså snakk om en konvertering fra en type til en annen, og man kaller en "getter".

----

## Hva forteller dette API-et oss?

```java
Instant now = Clock.systemUTC().instant();

long s = now.getEpochSecond();
int ns = now.getNano();
long ms = now.toEpochMilli();

// (alt kompilerer altså!)
```

----

<!-- .slide: data-state="instant-javadoc" -->

note:
Nye Java Time API har vektlagt å følge Java-_idiomer_:
- "get" eksponerer tilstand
- "to" gjør en konvertering til annen representasjon

Dette er idiomer som har eksistert siden tidenes morgen, men som til og med ikke de som har designet API-ene til Java selv har tatt ordentlig inn over seg.

---

## Ser saklig ut? ✅

<!-- .slide: data-transition="none-out" -->

```java
class Rune {
    boolean isTehAwesomest() {
        return true;
    }
}

var rune = new Rune();

assertThat(rune.isTehAwesomest());
```

----

<!-- .slide: data-transition="none" -->


## Ikke nå vel? 🤔

```java
class Rune {
    boolean isTehAwesomest() {
        return false; // 🙀
    }
}

var rune = new Rune();

assertThat(rune.isTehAwesomest());
```

----

<!-- .slide: data-transition="none-in" -->

## AssertJ 🙄

```java
class Rune {
    boolean isTehAwesomest() {
        return false; // 🙀
    }
}

var rune = new Rune();

assertThat(rune.isTehAwesomest()).isTrue();
```

note:
Ser fornuftig ut, semantikken stemmer når man leser, men har likevel ikke brukt API-et riktig.
Ingen indikasjon på feil bruk.


---


## <u>Enhetlige</u> API-er

Identifiser konvensjoner i egen kode

Sørg for å overholde de!




----

## <u>Enhetlige</u> API-er

```java
@Resource CustomerRepository customerRepository;

...

Customer customer = customerRepository.get(auth);

Cart cart = customer.startNewCart();
```

----

<!-- .slide: data-state="nullpointerexception" -->

## `NullPointerException`

##### `customer.startNewCart()`

note:
Får NPE når man bruker kundeobjektet man _tror_ man har fått.

----

## <u>Enhetlige</u> API-er

`get` vs. `find`

- `get` indikerer en forventning om at noe er tilgjengelig
- `find` indikerer at man prøver å få tak i noe man ikke er sikker på finnes

----

<!-- .slide: data-transition="none-out" -->

#### `.get(..) -> Customer`

```java
@Resource CustomerRepository customerRepository;

...

Customer customer = customerRepository.get(auth); // 💥

Cart cart = customer.startNewCart();
```

note:
Er man i en kontekst hvor det er forventet at kunden finnes? I så fall er det riktig å kalle `get`, og den skal kaste exception dersom kunden _ikke_ finnes. Da får du feilen der den skjer, og risikerer ikke at `null` propageres videre, og man får en NullPointerException et annet sted, og man må tracke seg tilbake hvor det er _mulig_ at den kan ha oppstått.

----

<!-- .slide: data-transition="none" -->

#### `.find(..) -> Optional<Customer>`

```java
@Resource CustomerRepository customerRepository;

...

Customer customer = customerRepository
         .find(auth)         // Optional<Customer>
         .orElseGet(() -> customerRepository.create(auth));
Cart cart = customer.startNewCart();
```

note:
Vet man ikke om kunden eksisterer er det riktig å kalle en find-metode, og man skal da få en `Optional` tilbake, som tvinger deg til å håndtere muligheten for at den ikke finnes.

----

<!-- .slide: data-transition="none" -->

#### `.find(..) -> Optional<Customer>`

```java
@Resource CustomerRepository customerRepository;

...

Customer customer = customerRepository.find(auth).get(); // 💥
         

Cart cart = customer.startNewCart();
```

note:
med mindre man gjør sånn da.


----

<!-- .slide: data-state="nullpointerexception" data-transition="none-in slide-out"-->

## `NoSuchElementException`

note:
Og det er jo også å forvente, siden man gjør en `get` på noe som ikke finnes.

Neste slide bare hvis det er tid!

----

<!-- .slide: data-transition="slide-in fade-out" -->

##### Case: return tuple

```java
enum IdentifierType {
  PHONENUMBER, EMAIL

  static ? parseIdentifier(String identifier) {
    IdentifierType type = IdentifierType.valueOf(
                          identifier.split(":")[0]);
    String value = identifier.split(":")[1];
    return ?;
  }
}
```

----

<!-- .slide: data-transition="fade" -->

##### Case: return tuple

```java
enum IdentifierType {
  PHONENUMBER, EMAIL

  static ? parseIdentifier(String identifier) { .. }
}
```

##### Et annet sted:

```java
class MyIdentifier {
  IdentifierType type;
  String value;
  MyIdentifier(IdentifierType type, String value) {..}
}

String id1 = "PHONENUMBER:12345678";
String id2 = "EMAIL:me@example.com";

? identifier = IdentifierType.parseIdentifier(id1);
```

----

<!-- .slide: data-transition="fade" -->

##### Løsningsforslag: return tuple

```java
enum IdentifierType {
  PHONENUMBER, EMAIL

  static <R> R parseIdentifier(
        String identifier,
        BiFunction<IdentifierType, String, R> resultComposer) { 
    
    IdentifierType type = IdentifierType.valueOf(
                          identifier.split(":")[0]);
    String value = identifier.split(":")[1];

    return resultComposer.apply(type, value);
  }
}
```

----

<!-- .slide: data-transition="fade" -->

##### Løsningsforslag: return tuple

```java
enum IdentifierType {
  PHONENUMBER, EMAIL

  static <R> R parseIdentifier(
        String identifier,
        BiFunction<IdentifierType, String, R> resultComposer) {..}
}
```

##### Et annet sted:

```java
class MyIdentifier {
  IdentifierType type;
  String value;
  MyIdentifier(IdentifierType type, String value) {..}
}

String id1 = "PHONENUMBER:12345678";

MyIdentifier identifier = IdentifierType
              .parseIdentifier(id1, MyIdentifier::new);
```



---

# Vi er <u>alle</u> API-designere!

---

# Et <u>bra</u> API

følger kjente <u>idiomer</u>, og <u>leder</u> andre utviklere til å kode riktig. Når man har lært en del av et API, kjenner man også andre deler av det samme <u>enhetlige</u> API-et.

