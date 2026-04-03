# 🎥 HTC Vive Tracker + Unreal Engine — Camera Tracking Setup

Appunti operativi per il setup completo del tracciamento camera con HTC Vive Tracker in Unreal Engine 5 tramite LiveLink e SteamVR.

---

## Indice

- [Avviare SteamVR senza headset (Null Driver)](#avviare-steamvr-senza-headset-null-driver)
  - [Directory di Steam](#directory-di-steam)
  - [File del Null Driver](#file-del-null-driver)
  - [File di configurazione di SteamVR](#file-di-configurazione-di-steamvr)

---

## Avviare SteamVR senza headset (Null Driver)

Prima di tutto occorre configurare SteaVR, per l'uso senza visore. Coi sono alcuni passaggi da efefttuare:
Prima di tutto, è necessario abilitare il **null driver** di SteamVR. Occorre modificare due file: il `default.vrsettings` del null driver e il `default.vrsettings` di SteamVR.

### Directory di Steam

Entrambi i file si trovano nella directory di installazione di Steam.

**Windows**

Di solito si trova in: `C:\Program Files (x86)\Steam\`
A meno che Steam non sia stato installato su un altro disco.

**Linux**

Dipende da come è stato installato Steam:
- Installato dal sito ufficiale (es. Ubuntu): `~/.local/share/Steam`
- Installato dal package manager ufficiale: `~/.steam`

---

### File del Null Driver

Il file si trova in: `[Steam Directory]/steamapps/common/SteamVR/drivers/null/resources/settings/default.vrsettings`

Aprire il file e sostituire: `"enable": false,` con: `"enable": true,`.


### File di configurazione di SteamVR

Il secondo file si trova in: `[Steam Directory]/steamapps/common/SteamVR/resources/settings/default.vrsettings`

Effettuare le seguenti tre modifiche:

| Valore originale | Valore da impostare |
|---|---|
| `"requireHmd": true,` | `"requireHmd": false,` |
| `"forcedDriver": "",` | `"forcedDriver": "null",` |
| `"activateMultipleDrivers": false,` | `"activateMultipleDrivers": true,` |
| `"turnOffScreensTimeout": 5.0,` | `"turnOffScreensTimeout": 0.0,` |

L'ultimo di questi si trova nella sezione Power, normalmente è settato a 5.0 di default.

> ⚠️ **Attenzione:** come indicato in cima al file stesso, questo file viene **sovrascritto ad ogni aggiornamento di SteamVR**. Per rendere le modifiche persistenti, potrebbe essere possibile inserire le stesse impostazioni nel file: `steamvr.vrsettings` che si trova nella cartella `config` di Steam. Assicurarsi di inserirle sotto l'intestazione `steamvr`. DA VERIFICARE perchè spesso il file in questione viene sovrascritto se si cambia qualcosa in SteamVR.
Consiglio di creare una copia di backup del file di configurazione descritto sopra da sostituire in caso di aggiorbamento di SteamVR

---

## Configurazione SteamVR — Associare il Tracker

Prima di avviare Unreal, il tracker deve essere associato e configurato correttamente in SteamVR.

**1. Associare il dispositivo**

Nel menu di SteamVR: `Dispositivi > Associa controller` 
Seguire la procedura guidata per il pairing del tracker.

**2. Assegnare il ruolo del tracker**
nel menù `Controller > Gestisci tracker`  Impostare il ruolo del tracker su **"Telecamera"**.

---

## Configurazione di Unreal Engine

### Plugin da attivare

Aprire windows > virtual production > live link > add source E aggiungere il live linkModifica > Plugin` e assicurarsi che i seguenti plugin siano **abilitati**:

| Plugin | Note |
|---|---|
| `OpenXR` | Base per il tracciamento XR |
| `OpenXRViveTracker` | Plugin per i tracker VIVE |
| `LiveLink` | Sistema di streaming dati live |
| `LiveLinkXR` | Bridge tra SteamVR/OpenXR e LiveLink |

> ⚠️ Riavviare il progetto dopo aver abilitato i plugin.

### Aggiungere la sorgente LiveLink (ogni apertura del progetto)
`windows > virtual production > live link > add source` E aggiungere il live link

Questa operazione va **ripetuta ogni volta** che si apre il progetto.

---

## Preparazione della Camera Virtuale in Unreal Engine

### 1. Creare l'attore Tracker

1. Creare un nuovo **Actor** vuoto nella scena e rinominarlo `Tracker`
2. Nel pannello **Details**, aggiungere un componente **`LiveLink Controller`**
3. Nel componente LiveLink Controller, impostare:
   - **Subject Representation**: selezionare il tracker dalla lista (deve essere attiva la sorgente LiveLinkXR)
   - **Controller Type**: `LiveLink Transform Controller`
4. Una volta configurato correttamente, i valori di **Transform** (posizione e rotazione) nell'attore saranno **in sola lettura** — significa che l'attore sta ricevendo dati live dal tracker fisico.

---

### 2. Creare l'attore TrackingOrigin

1. Creare un altro **Actor** vuoto e rinominarlo `TrackingOrigin`
2. Questo attore serve come **sistema di riferimento**: mette in relazione le coordinate reali del tracker con quelle della scena Unreal
3. Posizionando e ruotando `TrackingOrigin` nella scena, si definisce **dove si trova la camera nel mondo virtuale** e qual è il suo orientamento di partenza

> 💡 È l'attore che si sposta per "piazzare" la camera nel set virtuale — non si tocca mai direttamente il Tracker.

---

### 3. Creare la CineCamera

1. Aggiungere nella scena un **`CineCameraActor`**
2. Configurarla in modo che corrisponda alla camera reale utilizzata sul set:

| Parametro | Dove si trova | Note |
|---|---|---|
| **Focale** | `Current Focal Length` | In millimetri, es. `35mm` |
| **F-Stop / Diaframma** | `Current Aperture` | Es. `2.8` |
| **Dimensioni sensore** | `Filmback > Sensor Width / Height` | Es. `36 × 24mm` per Full Frame |

> ⚠️ Questi valori devono corrispondere esattamente alla camera fisica per avere una prospettiva coerente nel compositing.

---

### 4. Definire la gerarchia degli attori

Impostare la seguente gerarchia **parent > child** nel pannello **Outliner**:
```
TrackingOrigin
    └── Tracker
            └── CineCameraActor
```

- Trascinare `Tracker` su `TrackingOrigin` per renderlo figlio
- Trascinare `CineCameraActor` su `Tracker` per renderlo figlio

In questo modo i movimenti del tracker fisico vengono trasmessi alla camera virtuale, relativamente all'origine definita da `TrackingOrigin`.

---

### 5. Calibrazione dell'offset camera e correzione delle rotazioni

Questo è il passaggio più delicato e richiede pazienza.

**Offset di posizione (montaggio fisico del tracker)**

Misurare fisicamente sulla camera reale la distanza tra:
- Il **centro ottico della lente** (o il piano del sensore)
- Il **punto di montaggio del tracker**

Riportare questi valori come **offset di Transform sulla CineCameraActor** rispetto al Tracker, nei tre assi X, Y, Z.

**Correzione delle rotazioni**

A seconda di come il tracker è montato fisicamente sulla camera, potrebbe essere necessario correggere la rotazione. Procedere per tentativi, ruotando la `CineCameraActor` o il `TrackingOrigin`:

- Se il tracker è montato **ruotato di 90°**, applicare la correzione corrispondente
- Se il sistema usa un'orientazione **Y-up** (come capita con alcuni driver SteamVR) invece di **Z-up** (che è il sistema di Unreal), correggere ruotando il `TrackingOrigin` di **90° sull'asse X** (o il corrispondente necessario)

> 💡 **Metodo consigliato**: muovere la camera reale in una direzione alla volta (solo avanti, solo su, solo a destra) e verificare che la camera virtuale si muova nella stessa direzione. Correggere asse per asse fino ad allineamento completo.



*— Appunti in aggiornamento continuo*
