```mermaid
sequenceDiagram
    participant Controller as AXPSPServicePrincipalController
    participant Service as AXPSPServicePrincipalService
    participant API as AXPSPGraphAPI_GetEndDate
    participant Graph as Microsoft Graph API
    participant DB as SysAADClientTable
    participant Email as SysMailerMessageBuilder
    
    %% Start des Batch-Prozesses
    Controller->>DB: Lese Client-Daten
    Controller->>DB: Prüfe EndDateTime

    alt EndDateTime fehlt
        Controller->>Service: Fordere Service Principal Daten an
        Service->>API: Hole EndDateTime über API
        API->>Graph: Sende API-Request
        Graph-->>API: Antwort mit EndDateTime
        API-->>Service: Übergibt EndDateTime
        Service->>DB: Speichert EndDateTime
    end

    Controller->>DB: Prüfe verbleibende Tage
    alt Weniger als Grenzwert
        Controller->>Service: Lösen E-Mail-Benachrichtigung aus
        Service->>Email: Erstelle E-Mail-Nachricht
        Email->>Email: Setze Platzhalter in E-Mail-Vorlage
        Email->>Email: Senden an Benutzer
    end
    
    Controller->>Controller: Beende Batch-Prozess
