#  Pipeline Azure Data Factory — `progetto_profAI`

##  Panoramica
Questa pipeline di **Azure Data Factory (ADF)** implementa un flusso dati **bronze → silver → gold**, composto da:
1. una fase di **Data Wrangling** (dataflow di pulizia e trasformazione) che salva il dato in silver;
2. due attività **Get Metadata** per raccogliere informazioni sui file di input e intermediate (silver);
3. un’attività di **Copy** che scrive i dati nel livello *gold*, allegando i metadati del processo come proprietà del blob.

---

## ⚙️ Struttura generale

### Dataset coinvolti
| Dataset | Descrizione | Tipo |
|----------|--------------|------|
| `bronze_file` | File sorgente livello *bronze* | DelimitedText |
| `Csv_silver_modified` | Output del Data Flow (livello *silver*) | DelimitedText |
| `gold` | File di destinazione livello *gold* | DelimitedText |

---

## 🔄 Attività della pipeline

###  1. `DataWrangling` — *ExecuteDataFlow*
- Esegue il **dataflow `dataflow1`**.
- Responsabile della trasformazione e pulizia del file *bronze* → *silver*.
- Importa -> seleziona le colonne e cambia il nome delle intestazioni delle colonne -> converte la colonna "Valutazione" a Float -> filtra per film con Valutazione >= 7 -> li ordina in maniera decrescente sulla colonna Valutazione -> li salva nel contenitore silver.

---

###  2. `Get_Metadata_file_origine_bronze` — *GetMetadata*
- **Input**: dataset `bronze_file`
- **Campi letti**:
  - `columnCount`
  - `size`
  - `itemName`
- **Scopo**: recuperare dimensione, nome e caratteristiche del file *bronze* per scriverle come metadati nel file finale.

---

###  3. `Get_Metadata_file_from_silver` — *GetMetadata*
- **Input**: dataset `Csv_silver_modified`
- **Campi letti**:
  - `columnCount`
  - `size`
  - `lastModified`
- **Scopo**: ottenere informazioni sul file *silver* generato dal dataflow che verranno scritte nei metadati del file finale.

---

###  4. `Copy_from_silver_to_Gold` — *Copy Activity*
- **Input (source)**: dataset `Csv_silver_modified`
- **Output (sink)**: dataset `gold`


####  Metadati scritti nel blob di destinazione
| Nome metadato | Valore dinamico (ADF expression) |
|----------------|----------------------------------|
| `bronze_itemName` | `@{string(coalesce(activity('Get_Metadata_file_origine_bronze').output.itemName, 'unknown'))}` |
| `bronze_size` | `@{string(coalesce(activity('Get_Metadata_file_origine_bronze').output.size, 0))}` |
| `silver_columnCount` | `@{string(coalesce(activity('Get_Metadata_file_from_silver').output.columnCount, 0))}` |
| `silver_size` | `@{string(coalesce(activity('Get_Metadata_file_from_silver').output.size, 0))}` |
| `silver_lastModified` | `@{string(coalesce(activity('Get_Metadata_file_from_silver').output.lastModified, utcNow()))}` |

> Ogni valore è protetto da `coalesce()` e convertito con `string()` per evitare errori di tipo `Value cannot be null`.
---

## Risultato finale

Pipeline in Azure che prende in input un file dal contenitore bronze, lo elabora e poi lo salva nel contenitore contenitore gold. Il file finale provvisto dei metadati dei file di partenza (bronze e silver) avrà le intestazioni in italiano e conterrà solo i film con una valutazione superiore o uguale a 7.
---

## 🔗 Dipendenze e sequenza

```text
DataWrangling
   ├─▶ Get_Metadata_file_origine_bronze
   ├─▶ Get_Metadata_file_from_silver
           └─▶ Copy_from_silver_to_Gold








