# Robot Framework AI -demopohja

Pohja Robot Framework -testien tekemiseen tekoälyavustajan (Claude Code) avulla.
Avustaja käyttää [RobotMCP](https://github.com/manykarim/rf-mcp)-palvelinta:
se ajaa Robot Framework -avainsanoja oikeassa selaimessa askel kerrallaan ja
kokoaa toimivista askelista valmiin `.robot`-testitiedoston.

## Sisältö

| Työkalu | Tarkoitus |
| --- | --- |
| [Robot Framework](https://robotframework.org/) | Testien ajaminen |
| [Browser Library](https://robotframework-browser.org/) | Selaintestaus (Playwright) |
| [Robocop](https://robocop.readthedocs.io/) | Testien lint-tarkistus ja muotoilu |
| [RobotMCP (rf-mcp)](https://github.com/manykarim/rf-mcp) | MCP-palvelin, jonka kautta tekoäly ajaa ja rakentaa testejä |

```
.
├── .mcp.json          # RobotMCP-palvelimen määritys Claude Codelle
├── requirements.txt   # Python-riippuvuudet
├── tests/             # Testitiedostot (*.robot)
└── resources/         # Yhteiset avainsanat ja muuttujat (*.resource)
```

## Vaatimukset

- Python 3.10 tai uudempi
- Node.js 20 tai uudempi (Browser Library tarvitsee sen)
- [Claude Code](https://claude.com/claude-code) (CLI tai VS Code -laajennus)

## Käyttöönotto

### 1. Virtuaaliympäristö

Luo projektin juureen virtuaaliympäristö `.venv`. Nimen on oltava juuri tämä,
koska `.mcp.json` viittaa polkuun `.venv/Scripts/python.exe`.

**Windows (PowerShell):**

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
```

Jos PowerShell estää aktivointiskriptin, salli skriptit nykyiselle käyttäjälle:

```powershell
Set-ExecutionPolicy -Scope CurrentUser RemoteSigned
```

**macOS / Linux:**

```bash
python3 -m venv .venv
source .venv/bin/activate
```

### 2. Riippuvuudet

```bash
python -m pip install --upgrade pip
pip install -r requirements.txt
pip install rf-mcp
```

Asenna sen jälkeen Browser Libraryn selaimet ja Playwright:

```bash
rfbrowser init
```

### 3. RobotMCP Claude Codeen

Projektin juuressa oleva [.mcp.json](.mcp.json) määrittää RobotMCP-palvelimen,
joten erillistä asennusta ei tarvita:

```json
{
  "mcpServers": {
    "robotmcp": {
      "type": "stdio",
      "command": ".venv/Scripts/python.exe",
      "args": ["-m", "robotmcp.server"],
      "env": {}
    }
  }
}
```

macOS:llä ja Linuxilla vaihda `command`-kentän arvoksi `.venv/bin/python`.

1. Avaa Claude Code projektin juuressa.
2. Hyväksy `robotmcp`-palvelin, kun Claude Code kysyy sitä ensimmäisellä kerralla.
3. Tarkista yhteys komennolla `/mcp`: palvelimen `robotmcp` tilan pitäisi olla *connected*.

Jos palvelin ei käynnisty, kokeile käynnistää se käsin. Komennon pitää jäädä
odottamaan syötettä ilman virheilmoitusta (lopeta painamalla Ctrl+C):

```powershell
.\.venv\Scripts\python.exe -m robotmcp.server
```

## Testin tekeminen tekoälyllä

Kuvaa Claude Codelle testattava tilanne tavallisella kielellä, esimerkiksi:

> Tee testi, joka avaa sivun yle.fi, hyväksyy evästeet ja tarkistaa, että
> etusivulla näkyy uutislistaus. Tallenna testi tiedostoon tests/etusivu.robot.

Avustaja ajaa askeleet selaimessa RobotMCP:n kautta, korjaa epäonnistuneet
askeleet ja tallentaa lopuksi toimivan testin `.robot`-tiedostoon.

## Testien ajaminen

```bash
robot --outputdir results tests
```

Ilman näkyvää selainikkunaa (jos testit käyttävät `HEADLESS`-muuttujaa):

```bash
robot --outputdir results --variable HEADLESS:True tests
```

Tulokset tallentuvat `results/`-kansioon (`report.html`, `log.html`, `output.xml`).

## Koodin laatu

```bash
robocop check tests resources      # lint-tarkistus
robocop format tests resources     # automaattinen muotoilu
```

## Salaisuudet

Testattavan sivuston tunnukset annetaan ympäristömuuttujissa `SITE_USERNAME` ja
`SITE_PASSWORD`. Älä kirjoita tunnuksia testitiedostoihin äläkä liitä niitä
tekoälyavustajan keskusteluun.

### Vaihtoehto A: vain nykyiselle pääteistunnolle

Muuttujat ovat voimassa, kunnes suljet päätteen.

**Windows (PowerShell):**

```powershell
$env:SITE_USERNAME = "testikayttaja@example.com"
$env:SITE_PASSWORD = [Net.NetworkCredential]::new("", (Read-Host "Salasana" -AsSecureString)).Password
```

**macOS / Linux:**

```bash
export SITE_USERNAME="testikayttaja@example.com"
read -rsp "Salasana: " SITE_PASSWORD && export SITE_PASSWORD
```

### Vaihtoehto B: pysyvästi käyttäjän ympäristöön

**Windows (PowerShell):**

```powershell
[Environment]::SetEnvironmentVariable("SITE_USERNAME", "testikayttaja@example.com", "User")
$salasana = [Net.NetworkCredential]::new("", (Read-Host "Salasana" -AsSecureString)).Password
[Environment]::SetEnvironmentVariable("SITE_PASSWORD", $salasana, "User")
```

**macOS / Linux:** lisää `export`-rivit tiedostoon `~/.zshrc` tai `~/.bashrc`.

Ohjelmat näkevät pysyvät muuttujat vasta, kun ne käynnistetään uudelleen.

### Tunnukset testeissä

Robot Frameworkissa ympäristömuuttujaa luetaan syntaksilla `%{NIMI}`. Kirjoita
salasana kenttään Browser Libraryn `Fill Secret` -avainsanalla. Kun muuttujan
nimen eteen laitetaan pelkkä `%` ilman aaltosulkeita, salasana ei näy
`log.html`-lokissa:

```robotframework
*** Test Cases ***
Kirjautuminen
    Fill Text      id=username    %{SITE_USERNAME}
    Fill Secret    id=password    %SITE_PASSWORD
    Click          button[type=submit]
```

### Tekoälyavustaja ja tunnukset

RobotMCP-palvelin perii ympäristömuuttujat Claude Codelta. Jotta avustaja voi
ajaa kirjautumisaskeleita, aseta muuttujat (vaihtoehto A tai B) **ennen** kuin
käynnistät Claude Coden tai VS Coden. Jos asetit ne vasta myöhemmin, käynnistä
ohjelma uudelleen.

Pyydä avustajaa käyttämään testeissä muuttujia `%{SITE_USERNAME}` ja
`%SITE_PASSWORD`. Älä kirjoita oikeita tunnuksia keskusteluun.
