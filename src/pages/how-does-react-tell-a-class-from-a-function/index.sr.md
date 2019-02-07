---
title: Kako React razlikuje klase od funkcija?
date: '2018-12-02'
spoiler: Pričamo o klasama, new, instanceof, prototipskim lancima, i dizajniraju API-ja.
---

Pogledajmo ovu `Pozdrav` komponentu koja je definisana kao funkcija:

```jsx
function Pozdrav() {
  return <p>Ćao</p>;
}
```

Možemo je definisani i kao klasu:

```jsx
class Pozdrav extends React.Component {
  render() {
    return <p>Ćao</p>;
  }
}
```

([Doskoro](https://reactjs.org/docs/hooks-intro.html) je to bio jedini način da se koriste stvari kao `state`.)

Kad hoćete da renderujete `<Pozdrav />`, nije vas briga kako je on definisan:

```jsx
// Klasa ili funkcija? Kako god.
<Pozdrav />
```

Ali *samom React-u* svakako nije svejedno!

Ako je `Pozdrav` funkcija, React treba da je pozove:

```jsx
// Vaš kod
function Pozdrav() {
  return <p>Ćao</p>;
}

// React-ov kod
const rezultat = Pozdrav(props); // <p>Ćao</p>
```

Ali ako je `Pozdrav` klasa, React prvo mora da je instancira operatorom `new` i da *onda* pozove metodu `render` nad novokreiranom instancom:

```jsx
// Vaš kod
class Pozdrav extends React.Component {
  render() {
    return <p>Ćao</p>;
  }
}

// React-ov kod
const instance = new Pozdrav(props); // Pozdrav {}
const rezultat = instanca.render(); // <p>Ćao</p>
```

U oba slčaja, React-ov cilj je da dobije renderovan čvor (u ovom primeru je to `<p>Ćao</p>`). Ali tačni koraci zavise od toga kako je komponenta `Pozdrav` definisana.

**Kako onda React zna da li je nešto klasa ili funkcija?**

Kao što je bio slučaj sa [prethodnim postom]((/why-do-we-write-super-props/), ni ovaj **nije *obavezan* da savladate da biste bili produktivni pri upotrebi React-a**. Ja ovo godinama nisam znao. Molim vas, nemojte od ovoga da pravite pitanja na intervjuima. Zapravo, cela od priča se više tiče JavaSkripta nego React-a.

Ovo je priča za radoznalog čitaoca koji želi da nauči *zašto* React radi na određeni način. Jesi li to ti? Onda da istražimo zajedno.

**Ovo je dugo putovanje. Vežite se. Ova objava se sadrži mnogo informacija o samom React-u, ali ćemo proći kroz neke stvari o `new`, `this`, `class`, o streličastim funkcijama, o `prototype`, `__proto__`, `instanceof`, i kako sve te stvari zajedno rade u JavaSkriptu. Srećom, o svemu tome ne morate mnogo da razmišljate dok *koristite* React. E sad, ako ga implementirate...**

(Ako samo hoćete da znate odgovore, odskrolate do kraja.)

----

Prvo, moramo da razumemo zašto je važno da se različito ophodimo sa funkcijama i klasama. Obratite pažnju kako koristimo operator `new` kad zovemo klasu:

```jsx{5}
// Ako je Pozdrav funkcija
const rezultat = Pozdrav(props); // <p>Ćao</p>

// Ako je Pozdrav klasa
const instanca = new Pozdrav(props); // Pozdrav {}
const rezultat = instanca.render(); // <p>Ćao</p>
```

Hajde da otprilike vidimo šta operator `new` radi u JavaSkriptu.

---

Nekada davno, JavaSkript nije imao klase. Međutim, sličan šablon je mogao da se postigne korišćenjem običnih funkcija. **Konretno, možete da koristite *bilo koju* funkciju tako da liči na konstruktor klase ako dodate `new` ispred poziva:**

```jsx
// Obična funkcija
function Osoba(ime) {
  this.ime = ime;
}

var pera = new Osoba('Pera'); // ✅ Osoba {ime: 'Pera'}
var laza = Osoba('Laza'); // 🔴 Neće ovako
```

I dalje možete da pišete ovakav kod! Probajte u DevTools.

Ako ste pozvali `Osoba('Pera')` **bez** `new`, `this` iznutra bi pokazivao na nešto globalno i beskorisno (tipa `window` ili `undefined`). Kod bi stao da radi ili bismo uradili nešto smešno kao postavljanje `window.ime`.

Dodavanjem `new` ispred poziva, kažemo: „Hej, JavaSkripte, znam da je `Osoba` samo funkcija ali daj da se pravimo da je nešto slično konstruktoru klase. **Napravi objekat `{}` i neka `this` iz `Osoba` pokazuje na taj objekat da bismo mogli da vršimo dodele kao `this.ime`. A onda mi vrati taj objekat.**“

Eto šta operator `new` radi.

```jsx
var pera = new Osoba('Pera'); // Isti objekat kao `this` u `Osoba`
```

Operator `new` čini da sve što stavimo na `Osoba.prototype` bude dostupno u objektu `pera`:

```jsx{4-6,9}
function Osoba(ime) {
  this.ime = ime;
}
Osoba.prototype.reciĆao = function() {
  alert('Ćao, ja sam ' + this.ime);
}

var pera = new Osoba('Fred');
pera.reciĆao();
```

Ovako su ljudi emulirali klase pre nego što su one dodate u JavaSkript.

---

`new` je već neko vreme dostupno u JavaSkriptu. Ipak, klase su nešto novije. Daju nam da gornji kod napišemo tako da bude jasnije šta želimo da postignemo:

```jsx
class Osoba {
  constructor(ime) {
    this.ime = ime;
  }
  reciĆao() {
    alert('Ćao, ja sam ' + this.ime);
  }
}

let pera = new Osoba('Pera');
pera.reciĆao();
```

*Beleženje namere programera* je važno u jezicima i dizajniranju API-ja.

Ako napišete funkciju, JavaSkript ne može da nagađa da li treba da se zove kao `alert()` ili služi kao konstrutor pa se zove sa `new Osoba()`. Ako bismo zaboravili da dodamo `new` ispred poziva funkcije kao što je `Osoba`, dobili bismo čudne rezultate.

**Klasna sintaksa nam omogućava da kažemo: „Ovo nije bilo kakva funkcija. Ovo je klasa i ima konstrutor“.** Ako zaboravimo da joj prikačimo `new` pri pozivu, JavaSkript će se pobuniti:

```jsx
let pera = new Osoba('Pera');
// ✅  Ako je Osoba funkcija: radi
// ✅  Ako je Osoba klasa: opet radi

let laza = Osoba('Laza'); // Zaboravili smo new
// 😳 Ako je Osoba funkcija nalik konstruktoru: zbunjujuć rezultat
// 🔴 Ako je Osoba klasa: odmah puca
```

Ovo nam pomaže da ranije uhvatimo greške umesto da čekamo da se jave neki čudni bagovi (na primer, da se `this.ime` tretira kao `window.ime` umesto `george.ime`).

Međutim, ovo znači da React mora da doda `new` pre nego što pozove bilo koju klasu. Ne može samo da je pozove kao klasičnu funkciju, jer bi JavaSkript bacio grešku!

```jsx
class Brojač extends React.Component {
  render() {
    return <p>Ćao</p>;
  }
}

// 🔴 Ali React ne sme da uradi ovo:
const instanca = Brojač(props);
```

I eto nevolje.

---

Pre nego što pogledamo kako React rešava ovaj problem, važno je da se podsetimo da većina ljudi koji korise React koriste i kompilatore kao što je Babel da bi se otarasili modernih osobina (npr. klasa) zbog starijih brauzera. Zato moramo da vodimo računa da naš dizajn podržava i kompilatore.

U ranijim verzijama Babela, klase su mogle da se pozivaju i bez `new`. Međutim, ovo je ispravljeno — generisanjem dodatnog koda.

```jsx
function Osoba(ime) {
  // Blago pojednostavljeni izlaz iz Babela:
  if (!(this instanceof Osoba)) {
    throw new TypeError("Klasa se ne može pozvati kao funkcija");
  }
  // Our code:
  this.ime = ime;
}

new Osoba('Pera'); // ✅ Okej
Osoba('Laza');   // 🔴 Klasa se ne može pozvati kao funkcija
``` 

Možda ste videli takav kod u svojim generisanim fajlovima. Eto šta rade sve te funkcije `_classCallCheck` (proveri poziv klase). (Možete da smanjite veličinu generisanog koda ako uključite „loose mode“, tj. „labavi režim“, ali ovo može ukomplikovati kasniji prelezak na prave klase, ako se za taj korak odlučite).

---

Sada bi trebalo da vam je jasna glavna razlika između poziva sa `new` i bez `new`:

|  | `new Osoba()` | `Osoba()` |
|---|---|---|
| `class` | ✅ `this` je instanca `Osoba` | 🔴 `TypeError`
| `function` | ✅ `this` je instanca `Osoba` | 😳 `this` je `window` ili `undefined` |

Eto zašto je React-u važno da komponente pozove ispravno. **Ako je komponenta definisna klasa, React mora da iskoristi `new` za njen poziv.**

Znači React samo treba da proveri da li je nešto klasa ili ne?

Nije tako jednostavno! Čak i da možemo da [razlikujemo funkciju od klase u JavaSkriptu](https://stackoverflow.com/questions/29093396/how-do-you-check-the-difference-between-an-ecmascript-6-class-and-function), ovo i dalje ne bi radilo za klase generisane od strane alata kao što je Babel. Za brauzere su one same obične funkcije. Šteta za React.

---

Dobro, onda možda React da koristi `new` za svaki poziv? Nažalost, ni to neće uvek da upali.

Kada se obične funkcije pozovu sa `new`, kreira se objekat na koji pokazuje njihov `this`, To je poželjno za funkcije koje su napisane kao konstruktori (kao naša `Osoba` gore), ali bi bilo čudno za funkcije koje predstavljaju komponente:

```jsx
function Pozdrav() {
  // Ovde ne bismo očekivali da this bude bilo kakva instanca
  return <p>Ćao</p>;
}
```

Doduše, ovome bismo i mogli da progledamo kroz prste. Ali postoje još dva *druga* razloga zbog kojih ova ideja otpada.

---

Prvi razlog zašto uvek koristiti `new` neće da upali je to što, za nativne streličaste funkcije (ne one koje su proizvod kompilacije Babela), pozivi sa `new` bacaju izuzetak:

```jsx
const Pozdrav = () => <p>Ćao</p>;
new Pozdrav(); // 🔴 Pozdrav nije konstruktor
```

Postoji razlog zašto ovo funkcioniše ovako i posledica je dizajna streličastih funkcija. Jedna od njihovih glavnih prednosti je to što *nemaju* svoj `this`; umesto toga, `this` se uzima iz najbliže obične funkcije:

```jsx{2,6,7}
class Prijatelji extends React.Component {
  render() {
    const prijatelji = this.props.prijatelji;
    return prijatelji.map(prijatelj =>
      <Prijatelj
        // `this` se uzima iz metode `render`
        veličina={this.props.veličina}
        ime={prijatelj.ime}
        key={prijatelj.id}
      />
    );
  }
}
```

Okej, znači **streličaste funkcije nemaju svoj `this`.** Ali to znači da bi bile potpuno beskorisne kao konstruktori!

```jsx
const Osoba = (ime) => {
  // 🔴 Ovo nema smisla!
  this.ime = ime;
}
```

Prema tome, **JavaSkript ne dozvoljava da se streličaste funkcije pozovu sa `new`.**. Ako uradite to, verovatno ste svakako negde pogrešili, i bolje je da vam se kaže ranije. Iz istih razloga nije dozvoljeno ni da pozovete klasu *bez* `new`.

Ovo je fino ali nam i malo kvari planove. React ne može da sa `new` pozove sve jer bi program pukao nailaskom na streličastu funkciju! Mogli bismo da probamo da odredimo da li je u pitanju streličasta funkcija tako što pogledamo da li im fali `prototype`, i da njih ne zovemo sa `new`:

```jsx
(() => {}).prototype // undefined
(function() {}).prototype // {constructor: f}
```

Ali ovo [ne radi](https://github.com/facebook/react/issues/4599#issuecomment-136562930) za funkcije koje daje Babel. Možda ne deluje kao velika stvar, ali postoji još jedan razlog zašto je ovaj pristup ćorsokak.

---

Drugi razlog zašto ne možemo uvek da koristimo `new` je što React onda ne bi mogao da podržava komponente koje vraćaju stringove ili druge primitivne tipove.

```jsx
function Pozdrav() {
  return 'Ćao';
}

Pozdrav(); // ✅ 'Ćao'
new Pozdrav(); // 😳 Pozdrav {}
```

Opet, ovo ima veze sa smicalicama koje sa sobom vuče dizajn [operatora `new`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/new). Kako smo videli ranije, `new` govori endžinu JavaSkripta da napravi objekat, da taj objekat postane `this` unutar funkcije, i da nam kasnije vrati taj objekat kao rezultat.

Međutim, JavaSkript takođe dozvoljava funkciji koja se pozove sa `new` da *pregazi* povratnu vrednost poziva sa `new` tako što se vrati neki drugi objekat. Po svoj prilici, ovo je bilo korisno za šablone kao bazen objekata gde hoćemo da iznova upoterbljavamo instance:

```jsx{1-2,7-8,17-18}
// Lenje kreiranje
var nultiVektor = null;

function Vektor(x, y) {
  if (x === 0 && y === 0) {
    if (nultiVektor !== null) {
      // Koristi istu instancu
      return nultiVektor;
    }
    nultiVektor = this;
  }
  this.x = x;
  this.y = y;
}

var a = new Vektor(1, 1);
var b = new Vektor(0, 0);
var c = new Vektor(0, 0); // 😲 b === c
```

Međutim, `new` takođe *potpuno ignoriše* povratnu vrednost funkcije ako *nije* objekat. Ako vratite string ili broj, to je isto kao da uopšte niste napisali `return`.

```jsx
function Odgovor() {
  return 42;
}

Odgovor(); // ✅ 42
new Odgovor(); // 😳 Odgovor {}
```

Jednostavno ne postoji način da se pročita primitivna povratna vrednost (npr. broj ili string) iz funkcije koja je pozvana sa `new`. Zato, ako bi React uvek koristio `new`, ne bismo imali podršku za komponente koje vraćaju stringove!

To nije prihvatljivo, pa moramo da nađemo kompromis.

---

Šta smo dosad naučili? React mora da pozove klase (uključujući i one koje je generiaso Babel) *sa* `new` ali obične funkcije ili streličaste funkcije (uključujući i generiasne od strane Babela) *bez* `new`. I nema pouzdanog načina da ih razlikujemo.

**Ako ne možemo da rešimo opšti problem, da li možemo da rešimo uži?**

Kada definišete komponentu kao klasu, verovatno ćete da je izvedete iz `React.Component` da biste mogli da koristite ugrađene metode kao `this.setState()`. **Umesto da detektujemo sve klase, da li možemo da detektujemo samo klase izvedene iz `React.Component`?**

Da vam odmah kažem kraj priče: React radi baš tako.

---

Možda je način da se ispita da li je `Pozdrav` klasa koja predstavlja React component — ispitivanje da li važi `Freeting.protoype instanceof React.Component`:

```jsx
class A {}
class B extends A {}

console.log(B.prototype instanceof A); // true
```

Znam šta vam je u glavi. Šta je ovo?! Da bismo odgovorili na to pitanje, moramo da razumemo prototipe u JavaSkriptu.

Možda ste čuli za „prototipski lanac“. Svaki objekat u JavaSkriptu može da ima „prototip“. Kada napišemo `pera.reciĆao()`, ali objekat `pera` nema svojstvo `reciĆao`, to svojstvo tražimo na prototipu objekta `pera`. Ako ga tu ne nađemo, tražimo ga u sledećem prototipu u nizu; dakle, u prototipu prototipa objekta `pera`. I tako dalje.

**Možda je zbunjujuće, ali svojstvo `prototip` klase ili funkcije _ne_ pokazuje na prototip te vrednosti.** Ne zezam se.

```jsx
function Osoba() {}

console.log(Osoba.prototype); // 🤪 Nije Osobin prototip
console.log(Osoba.__proto__); // 😳 Osobin prototip
```

Prema tome, „prototipski lanac“ mu pre dođe `__proto__.__proto__.__proto__` nego `protoype.prototype.prototype`. Prošle su godine dok sam skapirao.

Šta je onda svojstvo `prototype` na funkciji ili klase? **To je `__proto__` koji se daje svim objektima koji se `new`-uju tom klasom ili funkcijom!**

```jsx{8}
function Osoba(ime) {
  this.ime = ime;
}
Osoba.prototype.reciĆao = function() {
  alert('Ćao, ja sam ' + this.ime);
}

var pera = new Osoba('Pera'); // Postavlja `pera.__proto__` na `Osoba.prototype`
```

I taj `__proto__` lanac je kako JavaSkript traži svojstvo:

```jsx
pera.reciĆao();
// 1. Da li pera ima svojstvo reciĆao? Ne.
// 2. Da li pera.__proto__ ima svojstvo reciĆao? Da. Zovi ga!

pera.toString();
// 1. Da li pera ima svojstvo toString? Ne.
// 2. Da li pera.__proto__ ima svojstvo toString? Ne.
// 3. Da li pera.__proto__.__proto__ ima svojstvo toString? Da. Zovi ga!
```

U praksi, skoro nikad ne treba da se petljate sa `__proto__` direktno iz koda osim ako ne debagirate nešto vezano za prototipski lanac. Ako nešto učinite dostupnim na `pera.__proto__`, onda treba da ga stavite na `Osoba.prototype`. Ili je bar tako prvobitno bilo zamišljeno.

Brauzeri u početku nije ni trebalo da učine svojstvo `__proto__`  vidljivim, jer se ono smatralo unutrašnjim detaljem jezika. Ali neki brauzeri su dodali `__proto__` i onda je se od toga vremenom skarabudžio neki standard (ali sada umesto njega treba koristiti `Object.getPrototypeOf()`).

**Ali i dalje me mnogo buni što svojstvo koje se zove `prototype` ne daje prototip te vrednost** (na primer, `pera.prototype` je nedefinisano jer `pera` nije funkcija). Lično, mislim da je to najveći razlog zbog kojeg čak i iskusni programeri često ne shvate ispravno kako rade prototipi u JavaSkriptu.

---

Ovo je dugačak post, a? Čini mi se da smo negde na 80%. Držite se.

Znamo da, kad kažemo `obj.foo`, JavaSkript zapravo traži `foo` u `obj`, `obj.__proto__`, `obj.__proto__.__proto__`, i tako dalje.

Kad radite sa klasama, ovaj mehanizam vam nije direktno izložen, ali `extends` radi pomoću dobrog starog prototipskog lanca. Tako instanca naše React klase ima pristup metodama kao `setState`:

```jsx{1,9,13}
class Pozdrav extends React.Component {
  render() {
    return <p>Ćao</p>;
  }
}

let c = new Pozdrav();
console.log(c.__proto__); // Pozdrav.prototype
console.log(c.__proto__.__proto__); // React.Component.prototype
console.log(c.__proto__.__proto__.__proto__); // Object.prototype

c.render();      // Nađeno na c.__proto__ (Pozdrav.prototype)
c.setState();    // Nađeno na c.__proto__.__proto__ (React.Component.prototype)
c.toString();    // Nađeno na c.__proto__.__proto__.__proto__ (Object.prototype)
```

Drugim rečima, **kada koristite klase, `__proto__` lanac instance se „odražava“ u hijararhiji klasa:**

```jsx
// `extends` lanac
Pozdrav
  → React.Component
    → Object (implicitno)

// `__proto__` lanac
new Pozdrav()
  → Pozdrav.prototype
    → React.Component.prototype
      → Object.prototype
```

2 Chainz.

---

Pošto `__proto__` lanac prati hijerarhiju klasa, možemo da proverimo da li je `Pozdrav` izveden iz `React.Component` tako što krenemo od `Pozdrav.prototype` i onda pratimo njegov `__proto__` lanac:

```jsx{3,4}
// `__proto__` lanac
new Pozdrav()
  → Pozdrav.prototype // 🕵️ Krećemo odavde
    → React.Component.prototype // ✅ Eto ga!
      → Object.prototype
```

Srećom po nas, `x instanceof Y` radi upravo takvu pretragu. Prati `x.__proto__` lanac i traži `Y.prototype`.

Obično se koristi da bi se odredilo da li je nešto instanca klase:

```jsx
let pozdrav = new Pozdrav();

console.log(pozdrav instanceof Pozdrav); // true
// pozdrav (🕵️‍ Krećemo odavde)
//   .__proto__ → Pozdrav.prototype (✅ Eto ga!)
//     .__proto__ → React.Component.prototype 
//       .__proto__ → Object.prototype

console.log(pozdrav instanceof React.Component); // true
// pozdrav (🕵️‍ Krećemo odavde)
//   .__proto__ → Pozdrav.prototype
//     .__proto__ → React.Component.prototype (✅ Eto ga!)
//       .__proto__ → Object.prototype

console.log(pozdrav instanceof Object); // true
// pozdrav (🕵️‍ Krećemo odavde)
//   .__proto__ → Pozdrav.prototype
//     .__proto__ → React.Component.prototype
//       .__proto__ → Object.prototype (✅ Eto ga!)

console.log(pozdrav instanceof Banana); // false
// pozdrav (🕵️‍ Krećemo odavde)
//   .__proto__ → Pozdrav.prototype
//     .__proto__ → React.Component.prototype 
//       .__proto__ → Object.prototype (🙅‍ Nema ga!)
```

Ali podjednako dobro radi i da se proveri da li klasa nasleđuje drugu klasu:

```jsx
console.log(Pozdrav.prototype instanceof React.Component);
// greeting
//   .__proto__ → Pozdrav.prototype (🕵️‍ Krećemo ovde)
//     .__proto__ → React.Component.prototype (✅ Eto ga!)
//       .__proto__ → Object.prototype
```

Tako možemo da odredimo da li je nešto klasa koja predstavlja React komponentu ili samo obična funkcija.

---

Ali React ne radi tako. 😳

Jedan problem oko rešenja sa `instanceof` je to što ne radi kada ima nekoliko kopija React-a na istoj stranici, i komponenta koju proveravamo je izvedena iz kopije `React.Component` iz *drugog* React-a. Mešanje više kopija React-a u isti projekat je loše iz nekoliko razloga ali smo se trudili da izbegnemo probleme ako je to moguće. (Ali ćemo sa hukovima [možda morati](https://github.com/facebook/react/issues/13991) da primoramo da nema duplikata).

Jedna druga moguća heuristika je da se proveri prisustvo metode `render` na prototipu. Međutim, tada [nije bilo jasno](https://github.com/facebook/react/issues/4599#issuecomment-129714112) kako će se API komponente razvijati. Svaka provera ima cenu pa ne bi trebalo da ubacimo više od jedne. Ovo takođe ne bi radilo ako `render` bilo definisano kao metoda na instanci, što bi se desilo kada se koristi sintaksa svojstva klase.

Zato je React [dodao](https://github.com/facebook/react/pull/4663) poseban fleg osnovnoj komponenti. React proverava prisustvo tog flega, i tako zna da li je nešto klasa koja predstavlja React komponentu ili ne.

Prvobitno je fleg bio na samoj klasi `React.Component`:

```jsx
// React-ov kod
class Component {}
Component.isReactClass = {};

// Možemo da proverimo ovako
class Greeting extends Component {}
console.log(Greeting.isReactClass); // ✅ Jeste
```

Međutim, enke implementacije klasa koje smo želeli da podržimo [nisu](https://github.com/scala-js/scala-js/issues/1900) kopirale statička svojstva (ili postavljane ne-standardni `__proto__`), pa se fleg gubio.

Zato je React [prebacio](https://github.com/facebook/react/pull/5021) ovaj fleg u `React.Component.prototype`:

```jsx
// React-ov kod
class Component {}
Component.prototype.isReactComponent = {};

// Možemo da proverimo ovako
class Greeting extends Component {}
console.log(Greeting.prototype.isReactComponent); // ✅ Jeste
```

**I to je bukvalno sve.**

Možda se pitate zašto je objekat a ne obična logička vrednost. Nije mnogo bitno u praksi, ali u ranijim verzijama Jest-a (pre nego što je bio Dobar™️) je automatsko mokovanje bilo uključeno po difoltu. Generisani mokovi nisu imali primitivna svojstva, pa [provera nije radila](https://github.com/facebook/react/pull/4663#issuecomment-136533373). Baš ti hvala, Jest.

Provera sa `isReactComponent` se i dan-danas [koristi u React-u](https://github.com/facebook/react/blob/769b1f270e1251d9dbdce0fcbd9e92e502d059b8/packages/react-reconciler/src/ReactFiber.js#L297-L300).

Ako ne klasu ne izvedete iz `React.Component`, React neće naći `isReactComponent` na prototipu, i neće tretirati komponentu kao klasu. Sada znate zašto [najizglasaniji odgovor](https://stackoverflow.com/a/42680526/458193) za grešku `Cannot call a class as a function` glasi da treba dodati `extends React.Component`. Kasnije je ubače[ubačeno upozorenje](https://github.com/facebook/react/pull/11168) koje se javlja kada postoji `prototype.render` ali ne `prototype.isReactComponent`.

---

Možda ćete reći da sam vas ovom pričom prvo namamio, a onda zeznuo. **Iskorišćeno rešenje je veoma prosto, ali sam okolišao da bih objasnio *zašto* je React došao do ovog rešenja i koje su alternative postojale.**

Prisećajući se sopstvenog isksutva, tako to obično ide sa API-jima biblioteka. Da bi API bio jednostavan za korišćenje, često morate da uzmete u obzir semantiku jezika (nekad i nekoliko jezika, kao i njihove buduće promene), performanse tokom izvršenja, produktivnost sa korakom kompilacije i bez njega, stanje ekosistema i načina na koji se dele paketi, rana upozorenja, i mnogo toga. Krajnji rezultat možda nije uvek najelegantniji, ali mora da bude najpraktičniji.

**Ako je konačna verzija API-ja uspešna, _njeni korisnici_ neće morati da razmišljaju o ovom procesu.** Umesto toga će moći da se usredsere na stvaranje aplikacija.

S druge strane, ako vas zanima... lepo je znati kako radi.
