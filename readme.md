#+Kurzdokumentation ActionHandler Neuronales Netz


#++Fachliche Beschreibung  


Fachlich wird das Neuronale Netz verwendet, um für eine Konten-Beschreibung das zugehörige Konto zu suchen. 

Als Input-Parameter wird bei der Abfrage eine Beschreibung übergeben. Hierzu wird mit dem Netz der „Best fit“ gesucht. Als Antwort wird die zugehörige SKR03, SKR04 und XBRL-Tag erwartet. 


Verwendung 
Zur Verwendung des Netzes gibt es drei Befehle:
 	* Erzeugen Neuronales Netz 
 	* Laden des Netzes 
 	* Abruf eines Kontos (per KI)
  Im Ergebnis liefert der ActionHandler das DATEV-Konto bzw. das XBRL-Tag, welches am ehesten zu der übergebenen Description passt. 

  Schritt 1: Erzeugen Neuronales Netz
   Dieser Schritt erzeugt einen neues Neuronales Netz, basierend auf den erzeugten Trainingsdaten, und legt dieses als Attachment im Graph ab. 

   Verwendeter Befehl 
   Folgender Befehl lässt den ActionHandler eine neue Trainingsdatei erzeugen: curl -X POST -H 'Content-type: application/json' --data-binary '{"capability": "retrain_model", "model":"unified"}' https://0.0.0.0:8443/request Technischer Ablauf Im Rahmen der Erstellung erfolgen folgende Schritte: 1.1	Graph auslesen (graphit.EESQuery) Der ActionHandler liest aus dem Graphen alle Knoten des Typs CompanySpecificCategorization, welche das Attribut AccountDescription besitzen (statische Mappings): graphit.EESQuery( 'ogit/_type:"ogit/Accounting/CompanySpecificCategorization"', '/AccountDescription:*' Es werden alle Attribute der Knoten gelesen und verwendet. Ein statisches Mapping besteht immer aus einem Konto, einer Beschreibung und entweder aus dem zugehörigen SKR03, SKR04, oder XBRL-Wert. 1.2	Maping-Tabelle (create_lookup_table) Zunächst wird ein statisches Mapping zwischen den verschiedenen Kontenrahemn erstellt. Hierzu werden statische CSV-Dateien verwendet (skr03_to_skr04.csv und skr04_to_xbrl.csv) 1.3	Trainingsdaten erzeugen (create_training_table) Mit Hilfe des statischen Mappings wird eine vollständiger Trainingsdatensatz erzeugt. Dieser besteht aus den folgenden „Spalten“: account, description, xbrl_tag, skr03, skr04 1.4	Pre-Processing (preprocess) Es folgt ein technischer Verarbeitungsschritt, in dem die Daten für das Neuronale Netz aufgearbeitet werden (Umwandlung Strings in Integers). In diesem Schritt werden auch die Meta-Daten für das Netz erzeugt. 1.5	Trainieren des Neuronalen Netzes (_train_model) Zuletzt erfolgt das eigentliche Trainieren des Netzes. Hierbei erfolgen diverse technische Schritte, die TensorFlow spezifische Funktionen verwenden. Zuletzt werden die erzeugten Trainingsdaten als ogit/Attachment im Graph abgelegt: attachment = graphit.Attachment(self.session, data={ 'ogit/_owner': 'de.ey.com', 'ogit/_type': 'ogit/Attachment', '/DataName': "CNNModel", '/ModelName': model_name, '/Status': 'production', '/Meta': json.dumps({'maxlen': trainset.maxlen, 'xbrl_map': trainset.xbrl_map, 'skr03_map': trainset.skr03_map, 'skr04_map': trainset.skr04_map}), '/MapTable': json.dumps(mapping) }) Das erzeugte Neuronale Netz wird als Anhang zu dem erzeugten Knoten abgelegt. Schritt 2: Trainieren des Netzes (_load_model) Das neuste, im Graph vorhandene Neuronale Netz wird geladen und damit für zukünftige Abfragen verwendet. Verwendeter Befehl Folgender Befehl lässt den ActionHandler das neuste Netz laden: curl -X POST -H 'Content-type: application/json' --data-binary '{"capability": "reload_model", "model":"unified"}' https://0.0.0.0:8443/request Technischer Ablauf 2.1	Graph auslesen Es werden alle ogit/Attachment gelesen: q = graphit.EESQuery(             'ogit/_type:"ogit/Attachment"',             '/DataName:"CNNModel"',             f'/ModelName:"{model}"',             '/Status:"production"',             '/Meta:*',             '/MapTable:*' ) 2.2	Auswahl des neusten Knotens (newest_model) Alle gefunden Modelle werden nach ogit/_modified-on sortiert und der neuste wird ausgewählt. 2.3	Laden des Netzes Es erfolgen diverse technische Schritte, die TensorFlow spezifische Funktionen verwenden. 

Schritt 3: Abruf eines Kontos 

Aufruf via KI (geht ebenfalls gegen den oben genannten Endpunkt „request"): exit, datev, datev_prob, xbrl, xbrl_prob = action( capability: "predict", model: "unified", description: "…", kontenrahmen: "…", category: "…", xbrl_tag: "…" ) 

Beispiel eines Aufrufes aus einer KI: do xbrl: LOCAL::ACCOUNT, xbrl_prob: LOCAL::CERTAINTY = action( capability: "predict", description: "${ogit/name}" ) set(XRBLTag = LOCAL::ACCOUNT) set(XRBLTagCertainty = LOCAL::CERTAINTY) 

Der ActionHandler selbst ist wie folgt konfiguriert:
''' "handlers": [{ name": "PredictionActionHandler", "capability": "predict", "implementation": "REST", "applicability": ["on ogit/_id"], "url": "https://0.0.0.0:8443/request” "verify_host": "false" }] '''