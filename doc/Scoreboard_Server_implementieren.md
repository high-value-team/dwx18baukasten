# Scoreboard Server implementieren
Die Aufgabe des Scoreboard Server ist es, Spielergebnisse potenziell vieler Game Server zu verwalten. Ergebnisse können ihm für Spiele gemeldet werden, deren Spieler bei ihm registriert sind.

Der Scoreboard Server ist einfacher zu implementieren als der Game Server. Es gibt weniger Endpunkte und keine Abhängigkeit zu einem anderen Server.

## Endpunkte
### High Scores
Der Scoreboard Server führt eine Tabelle aller Spiele und gibt auf Wunsch die Top 10 Liste der gewonnenen Spiele zurück.

**Request** 

`GET {{sb_address}}/api/highscores`

**Response**

* Statuscode: 200
* Content-Type: application/json
* Siehe `hm.contracts.data.dto.scoreboard.HighScoreDto{}`
* Beispiel:

```
[
	{
		"name": "Peter",
		"value": 686
	},
	...
]
```

### Spieler registrieren
Bevor Spielresultate beim Scoreboard registriert werden können, muss der Spieler sich dort anmelden. Es wird für ihn eine Spieler-ID generiert und zurückgegeben. Die ist beim späteren Registrieren von Spielergebnissen anzugeben.

**Request**

`POST {{sb_address}}/api/players?name={name}`

**Response**

* Statuscode: 200
* Content-Type: text/plain
* Beispiel: `80723a94-b4ff-478c-a326-1872823ad482`

### Score registrieren
Wenn Spiele nicht abgebrochen werden, wird ihr Ergebnis dem Scoreboard (vom Game Server) gemeldet.

**Request**

`POST {{sb_address}}/api/players/{playerId}/scores`

* Content-Type: application/json
* Siehe `hm.contracts.data.dto.ScoreDto{}`
* Beispiel:

```
{
	"wordlength":10,
	"numberoftrials":7,
	"numberoffailedtrials":3,
	"gamewaswon":true
}
```

**Response**

* Statuscode: 200


## Der Baustein
### Request Handler
Der Controller des Scoreboard Servers delegiert Anfragen an `hm.scoreboard.ScoreboardRequestHandler{}`. Deren Schnittstelle sieht so aus:

```
public class ScoreboardRequestHandler
{   
    public ScoreboardRequestHandler(ScoreRepository repo)

    public string RegisterPlayer(string name)
    public void RegisterScore(string playerId, ScoreDto score
    public HighScoreDto[] QueryHighScores()}
```

Zu beachten ist die Abhängigkeit vom `ScoreRepository{}`, die im Ctor injiziert wird.

### Score Repository
Das Score Repository `hm.scoreboard.providers.ScoreRepository{}` speichert die Spielergebnisse auf der lokalen Festplatte (im Container) in JSON-Dateien. Seine Schnittstelle ich nicht relevant, aber für die Instanziierung im Controller oder in `Main()` hier der Ctor:

```
public class ScoreRepository
{
    public ScoreRepository() : this(DEFAULT_PATH) {}
    public ScoreRepository(string path)
    ...
} 
```
### Konfiguration
Der Server kann über die Kommandozeile oder Umgebungsvariablen konfiguriert werden. Wie das beim Aufruf konkret geschieht, kapselt `hm.scoreboard.providers.CLI{}`.

Ein `CLI{}`-Objekt sollte bei Start zum Beispiel in `Main()` sofort instanziiert werden.

```
public class CLI
{
    public CLI(string[] args)
        
    public Uri Address { get; }
}
```

**Usage**

* Default: `scoreboardservice.exe`
  * Die Adresse wird auf `http://localhost:9000` gesetzt.

* Adresse explizit setzen: `scoreboardservice.exe -address http://localhost:9001`

**Umgebungsvariablen**

Alternativ kann die Adresse, auf der der Server erreichbar ist, auch über die Umgebungsvariable `HM_SCOREBOARD_SERVICE_ADDRESS` gesetzt werden.

## Der Server
Der Server wird mit der konfigurierten (oder einer festen) Adresse gestartet wie [in der Einführung beschrieben](../README.md). Wenn Konfiguration gewünscht ist, dann am besten via `CLI{}`.

Wenn der Controller in derselben Assembly wie der Aufruf von `ServiceHost.Run()` liegt, muss dessen Typ als Parameter explizit übergeben werden.

Außerdem ist zu entscheiden, wo/wann der Request Handler und das Score Repository instanziiert werden. Geschieht das bei jedem Aufruf des Controllers erneut oder soll immer wieder dieselbe Instanz verwendet werden?

Ist Letzters gewünscht, kann die Request Handler Instanz leider nicht dem Controller per Injektion bekannt gemacht werden. Es ist also ein anderer Weg über statische Daten zu finden.

## Der Controller
Der Controller implementiert für jeden Endpunkt eine Funktion, die den Aufruf an den ihm bekannten Request Handler weiterleitet. Beispiel:

```
[EntryPoint(HttpMethods.Get, "/api/highscores")]
public HighScoreDto[] QueryHighScores() {
    Console.WriteLine("GET QueryHighScores");
    return _handler.QueryHighScores();
}
```

Bei korrektem Einsatz der ServiceHost-Attribute ist kein (De)Serialisierungsaufwand zu treiben, auch wenn Daten im JSON-Format übertragen werden.

Aufgabe des Controllers ist es nur, den Request Handler via HTTP verfügbar zu machen. Der Controller sollte daher keine weitere Logik enthalten.

## Testen
Der Server kann auch ohne Client mit einem Tool wie `curl`, Insomnia oder Postman getestet werden.

![](images/insomnia.png)

Ein wenig aufpassen musst du dabei nur, wenn in der URL eines Aufrufs das Ergebnis eines anderen benutzt wird. Im Bild ist es z.B. die Spieler-ID, die zuerst durch Registrierung des Spielers generiert werden muss.
