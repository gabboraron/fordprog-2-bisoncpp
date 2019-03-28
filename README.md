# Bisonc++
> minta fájlok: http://deva.web.elte.hu/fordprog/bisonc++.zip

## Nyelvtanírás alapjai
- A kezdőszimbólum neve `start`
- A *terminálisok (tokenek) nagybetűsek*, a *nemterminálisok kisbetűsek*.
- A szabály bal- és jobb oldalát `:` választja el egymástól
- Az alternatívák között `|` szerepel.
- A szabályalternatívák sorozatát `;` zárja le.
- `C++` stílusú meg jegyzések írhatók a szabályokhoz.
- Az ε-t üres szabályjobb oldal valósítja meg, a gyakorlatban egy `//ures` megjegyzést szokás írni helyette.
- A jobb oldalak után `{` és `}` között `C++` kód írható, ami mindannyiszor végrehajtó dik, amikor az adott szabályt az elemző használja.
Ezekszerint: 
*S -> aC|C*

*C -> ε|bC*
nyelvtannak ez feletethető meg:
````Lex
start:
A c
|
	c
;
````

````Lex
c:
	//ures
|
	B c
;
````


## 1

### Opciók
`%baseclass-preinclude <iostream>`
A generálandó osztályhierearchia ősosztályába beilleszti az iostream fej-állományt. Ez azokban a példákban lesz majd fontos, ahol a szabályokhoz csatolt akciókban írni akarunk a standard outputra.

### Tokenek
`%token ELEM NYITO CSUKO VESSZ`
A tokentípusokat a `%token` direktíva segítségével definiáljuk. A nyelvtan terminálisai az itt felsorolt négy elem, melyekből a `Bisonc++` az általa generálandó `Parser` osztályba egy felsorolási típust fog létrehozni.

> A nyelvtan egy zárójelbe tett, vesszővel elválasztott elemekből álló lista szintaxisát adja meg.

fájl: `1/lista.y`
````Yacc
%baseclass-preinclude <iostream>

%token ELEM NYITO CSUKO VESSZO
%%

start:
	NYITO lista CSUKO
;

lista:
	// ures
|
	ELEM folytatas
;

folytatas:
	// ures
|
	VESSZO ELEM folytatas
;
````

---

### Hozzá tartozó Flex
Ez egy `Flex` forrásfájl, melyről részletes leírás itt található: http://deva.web.elte.hu/fordprog/flex-help.pdf A szintaktikus elemzés számára fontos részletek a következők:

`#include "Parserbase.h"`
Ezt a fejállományt a `Bisonc++` fogja generálni. Beillsztésével láthatóvá tesszük a `lista.y` fájlban megadott tokeneket

`return Parser::ELEM; `
Az egyes reguláris kifejezések sikeres illesztésekor a lexikális elemző vissza fog térni a megfelelő tokennel.

fájl: `1/lsita.l`
````Lex
%option noyywrap c++

%{
#include "Parserbase.h"
%}

BETU        [a-zA-Z]
SZAMJEGY    [0-9]
WS      [ \t\n]

%%

({BETU}|{SZAMJEGY})+    return Parser::ELEM;
","         return Parser::VESSZO;
"["         return Parser::NYITO;
"]"         return Parser::CSUKO;

{WS}+   // feher szokozok: semmi teendo

. {
    std::cerr << "lexikalis hiba" << std::endl;
    exit(1);
}

%%
````
---

### Hozzá tartozó C++
Ez a `C++` forrás tartalmazza a `main` függvényt,amelyben ellenőrizzük a paranancssori argumentum meglétét és megpróbáljuk megnyitni a megadott fájlt. Ha ez sikeres, akkor ezzel az inputtal létrehozunk egy szintaktikus elemző objektumot(`pars`), melynek `parse()` metódusával indítjuk el az elemzést.
fájl: `1/lista.cc`
```C++
#include <iostream>
#include <fstream>
#include <sstream>
#include "Parser.h"
#include <FlexLexer.h>

using namespace std;

int main( int argc, char* argv[] )
{
	if( argc != 2 )
	{
		cerr << "Egy parancssori argumentum kell!" << endl;
		return 1;
	}
	ifstream in( argv[1] );
	if( !in )
	{
		cerr << "Nem tudom megnyitni: " << argv[1] << endl;
		return 1;
	}
	
	Parser pars(in);
	pars.parse();
	return 0;
}
````

### Fordítás 1
- `cd 1`
- `flex lista.l`
- `bisoncic++ lista.y`
- ekkkor keletkezik: `Parserbase.h`,`Parser.ih`,`Parser.h`,`parse.cc`
- a `Parser.ih`,`Parser.h` nem kerül felülírásra legközelbb, így futtatható

### Parser.h
Ez a fejállomány definiálja a `Parser` osztályt. Ahhoz, hogy a szintaktikus elemző együtt tudjon működni a lexikális elemzővel, `include`-oljuk a `FlexLexer.h` fejállományt, felvesszük a lexikális elemzőt a Parser osztály adattagjai közé (`lexer`), és hozzáadunk az osztályhoz egy konstruktort, ami a kap ott bemeneti adatfolyammal inicializálja a `lexer`t. (Ezt a konstruktort hívtuk meg a `main` függvényben.)
fájl: `1/Parser.h`

````C++
// Generated by Bisonc++ V4.09.02 on Thu, 24 Sep 2015 17:14:55 +0200

#ifndef Parser_h_included
#define Parser_h_included

// $insert baseclass
#include "Parserbase.h"
#include "FlexLexer.h"
#include <cstdlib>


#undef Parser
class Parser: public ParserBase
{
        
    public:
        Parser( std::istream & in ) : lexer( &in, &std::cerr ) {}
        int parse();

    private:
        yyFlexLexer lexer;
        void error(char const *msg);    // called on (syntax) errors
        int lex();                      // returns the next token from the
                                        // lexical scanner. 
        void print();                   // use, e.g., d_token, d_loc

    // support functions for parse():
        void executeAction(int ruleNr);
        void errorRecovery();
        int lookup(bool recovery);
        void nextToken();
        void print__();
        void exceptionHandler__(std::exception const &exc);
};


#endif
````

### Parser.ih
Ebben az implementációs fejállományban az error tagfüggvény átírásával szabhatjuk testre a hibaüzeneteket. Ez a fejállomány definiálja továbbá a `Parser` osztály `lex()` függvényét: Valahányszora szintaktikus elemzőnek szüksége van a szöveg következő tokenjére, ezt a függvényt hívja meg. Ebben a példában ennek a függvénynek összesen annyi a teendője, hogy meghívja a `Parser` osztály adattag jai közé felvett lexikális elemző objektum `yylex()` metódusát,és a kapott eredményt adja vissza. Ez az eredmény az, amit a `Flex` forrásfájlban látható `return` utasítások adnak.

fájl: `1/Parser.ih`

````C++
// Generated by Bisonc++ V4.09.02 on Thu, 24 Sep 2015 17:14:55 +0200

    // Include this file in the sources of the class Parser.

// $insert class.h
#include "Parser.h"


inline void Parser::error(char const *msg)
{
    std::cerr << msg << '\n';
}

// $insert lex
inline int Parser::lex()
{
    return lexer.yylex();
}

inline void Parser::print()         
{
    print__();           // displays tokens if --print was specified
}

inline void Parser::exceptionHandler__(std::exception const &exc)         
{
    throw;              // re-implement to handle exceptions thrown by actions
}


    // Add here includes that are only required for the compilation 
    // of Parser's sources.



    // UN-comment the next using-declaration if you want to use
    // int Parser's sources symbols from the namespace std without
    // specifying std::

//using namespace std;
````

### Fordítás 2
- `g++ -o lista lista.cclex.yy.cc parse.cc`
- a frodítás eredménye a `lista` futtatható állomány, a szintaktikus elemző
- `./lista jo.txt`
- `./lista hibas.txt`
Fordíthatunk ehelyett a `make` parancssal is, a `Makefile`t használva.


## 2
Az előzőhöz képest `jo.txt` és `hibas.txt` fájlok alapján kitalálható, hogy milyen nyelvet szeretnénk elemezni.
A `2-hibakezeles` könyvtár tartalma azt mutatja meg, hogyan lehet jobb hibaüzenetet adni szintaktikus hiba esetén illetve ilyen esetben is tovább folytatni az elemzést.

- A Flex forrásfájlban az `yylineno` opció segítségével gondoskodunk róla, hogy a lexikális elemző számlálja a sorokat.
- A `Parser.ih` fájlban a `lex` függvényben a `lineno` metódussal kérjük el a lexikális elemzőtől az aktuális sorszámot, és ezt a `Parser` osztály `d_loc__` adattagjának egyik mezőjébe mentjük el.
- A `Parser.ih` fájlban definiált `error` függvényt úgy módosítjuk, hogy felhasználja ezt a helyinformációt.
- Az `fl.y` fájl nyelvtani szabályait kibővítjük úgy, hogy használja a speciális error nemterminális szimbólumot. Ha az elemző szintaktikus hibát észlel, akkor megpróbálja illeszteni az `error`-t tartalmazó hibaalternatívákat. Vigyázni kell arra, hogy mindig egy jól meghatározott terminális zárja le ezeket a hibaalternatívákat, különben könnyen konfliktusokat okoznak a nyelvtanban.
