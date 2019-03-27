# Bisonc++
> minta fájlok: http://deva.web.elte.hu/fordprog/bisonc++.zip

## 1

### Opciók
`%baseclass-preinclude <iostream>`
A generálandó osztályhierearchia ősosztályába beilleszti az iostream fej-állományt. Ez azokban a példákban lesz majd fontos, ahol a szabályokhoz csatolt akciókban írni akarunk a standard outputra.

### Tokenek
`%token ELEM NYITO CSUKO VESSZ`
A tokentípusokat a `%token` direktíva segítségével definiáljuk. A nyelvtan terminálisai az itt felsorolt négy elem, melyekből a `Bisonc++` az általa generálandó `Parser` osztályba egy felsorolási típust fog létrehozni.

> A nyelvtan egy zárójelbe tett, vesszővel elválasztott elemekből álló lista szintaxisát adja meg.

fájl: `1/lista.y`
````
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

fájl: `lsita.l`
````Flex
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

### Hozzá tartozó C++
Ez a `C++` forrás tartalmazza a `main` függvényt,amelyben ellenőrizzük a paranancssori argumentum meglétét és megpróbáljuk megnyitni a megadott fájlt. Ha ez sikeres, akkor ezzel az inputtal létrehozunk egy szintaktikus elemző objektumot(`pars`), melynek `parse()` metódusával indítjuk el az elemzést.
fájl: `lista.cc`
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
