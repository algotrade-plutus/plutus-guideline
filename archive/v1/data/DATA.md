# Data Guide
A Guide to the data for the Plutus project

The datasets linked below are **Tier 2 raw sources** in the V2 PLUTUS data-tier
model: they are declared under `data_sources.raw` in the manifest with
`kind: google_drive`, and the verifier downloads them before executing the
pipeline steps that `satisfies` lists. See [README §3](../README.md#3--data-tiers)
for the full four-tier model and the complete `_DATA_SOURCE` field reference.

## Backtest Data Suite
- Can be accessed [here](https://drive.google.com/drive/folders/11_qu8XYs1rnuhN4QfUEgFnIA3qqifQrZ)
- Contains:
    - Offline Market Data (Tick) Pre 2023
    - Financial Report Pre 2023

### Data Field
- Details can be found in [this document](./data-field-description.md).

## Sample Project Data
- Data of the two sample projects can also be explored [here](https://drive.google.com/drive/folders/1V2mZK20JWXRuEVHDoOad4cEyzqZs72CY):
    - [Deep Market Maker](https://drive.google.com/drive/folders/1Df2NTc0NHEfjdCtFMAdJ-k4_X7ODS5uC)
    - [Smart Beta](https://drive.google.com/drive/folders/11YOHtzsivwJsOG23PbCyhlJjxroaR_zP)
