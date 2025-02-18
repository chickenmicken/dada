```mermaid
sequenceDiagram
    autonumber
    participant Controller as AXPSPServicePrincipalController
    participant Service as AXPSPServicePrincipalService
    participant API as AXPSPGraphAPI_GetEndDate
    participant GraphAPI as AXPGraphAPI
    participant Graph as Microsoft Graph API
    participant DB as SysAADClientTable
    participant Contract as AXPSPServicePrincipalContract
    participant Config as AXPSPParameters
    participant Email as SysMailerMessageBuilder

    %% Start des Batch-Prozesses
    Controller->>DB: Lese alle Service Principals aus SysAADClientTable
    loop Für jeden Eintrag in DB
        Controller->>DB: Hole ClientId & EndDateTime
        
        %% Überprüfung, ob EndDateTime gesetzt ist
        alt EndDateTime nicht gesetzt
            Controller->>Service: Fordere EndDateTime über API an (ClientId)
            Service->>API: Anfrage für EndDateTime mit ClientId
            API->>GraphAPI: Rufe getAuthToken() für Authentifizierung
            GraphAPI-->>API: Bearer Token zurückgegeben
            API->>Graph: HTTP GET Request mit Token & ClientId
            Graph-->>API: JSON-Antwort mit EndDateTime
            API-->>Service: Extrahiere EndDateTime
            Service->>Contract: Erstelle Contract mit ClientId & EndDateTime
            Contract-->>Service: Contract mit Daten zurückgegeben
            Service->>DB: Speichere EndDateTime in SysAADClientTable
        else EndDateTime bereits gesetzt
            Controller-->>Controller: Keine API-Abfrage nötig
            DB->>Contract: Lade vorhandene Werte in Contract
        end

        %% Berechnung der verbleibenden Tage
        Controller->>DB: Berechne RemainingDays = EndDateTime - Heute
        Controller->>Config: Hole Warnschwellenwerte (RemainingDaysRed, RemainingDaysYellow)
        DB->>Contract: Aktualisiere Contract mit RemainingDays

        %% Prüfung, ob eine Benachrichtigung notwendig ist
        alt RemainingDays <= RemainingDaysRed
            Controller->>Service: Lösen E-Mail-Benachrichtigung aus
            Service->>Email: Erstelle E-Mail für ClientId
            Email->>Config: Hole E-Mail-Vorlage & Empfänger aus AXPSPParameters
            Contract->>Email: Übergibt ClientId, EndDateTime, RemainingDays
            Email->>Email: Setze Platzhalter in E-Mail
            Email->>Email: Sende E-Mail an Empfänger
        else RemainingDays <= RemainingDaysYellow
            Controller-->>Controller: Nur visuelle Warnung, keine E-Mail
        else Mehr als Grenzwert
            Controller-->>Controller: Keine Aktion notwendig
        end
    end
    
    %% Abschluss des Batch-Prozesses
    Controller->>Controller: Beende Batch-Prozess
