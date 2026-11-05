# DataWorks Challenge 2026

## Reservoir history matching and production forecasting

Team Data Wizards: Hakim, Harith, Farish, and Imran  
Submission date: 16 October 2026

## Project overview

This project uses simulated reservoir cases and observed field production to estimate uncertain reservoir parameters, compare two proxy-modeling approaches, and forecast production from April 2008 through January 2028.

The study contains 100 simulated cases, each defined by four uncertain parameters:

- Fault transmissibility
- Porosity multiplier
- Permeability multiplier
- Aquifer pore volume

The observed and simulated production histories are aligned to 41 quarterly dates from 1 January 1998 to 1 January 2008. Oil, gas, and water rates are modeled together with their cumulative-production counterparts.

## Project contents

The project consists of the notebook and the files in `Dataset`. Other files and folders in the directory are not required for the project.

| File | Description |
|---|---|
| `Dataworks.ipynb` | Main notebook containing data preparation, exploratory analysis, model training, history matching, forecasting, and validation. |
| `Dataset/01_Introduction.parquet` | Reservoir nomenclature and the four uncertain parameters for the cases. |
| `Dataset/02_Field_Production.parquet` | Observed field-production data and production-series information. |
| `Dataset/03_Train_Cases_1.parquet` | Training cases 1–20. |
| `Dataset/04_Train_Cases_2.parquet` | Training cases 21–40. |
| `Dataset/05_Train_Cases_3.parquet` | Training cases 41–60. |
| `Dataset/06_Train_Cases_4.parquet` | Training cases 61–70. |
| `Dataset/07_Validation_Cases.parquet` | Validation cases 71–85. |
| `Dataset/08_Test_Cases.parquet` | Test cases 86–100. |
| `Dataset/09_Template_Deliverable.parquet` | Template structure for the ten forecast cases. |
| `Dataset/models/reservoirLSTM_1.pth` | Saved trained ReservoirLSTM model weights used by the forecasting workflow. |
| `Dataset/clean/Template_filled.xlsx` | Completed forecast workbook containing the ten selected forecast cases. |

The notebook internally works with clean tables representing 100 cases, 4,100 simulated quarterly records, and 41 observed quarterly records. Intermediate tables are runtime artifacts, while the trained model and completed forecast workbook are included in `Dataset`.

## Notebook workflow

The notebook is organized into eight main stages:

1. **Load and validate data** – authenticates to the private data repository when required, checks the available source files, and verifies the case splits and table shapes.
2. **Prepare the time series** – standardizes column names, creates the four-parameter case table, aligns irregular simulator output to a quarterly grid, and derives observed cumulative values using trapezoidal integration.
3. **Exploratory data analysis** – examines parameter ranges and independence, compares simulated cases, and visualizes oil, gas, and water production profiles.
4. **Proxy-model design** – evaluates how the four static reservoir parameters can condition a sequential production model.
5. **LSTM proxy** – trains a two-layer PyTorch LSTM. The four static parameters are projected into the initial hidden and cell states, while the sequence input contains six production variables: oil, gas, and water rates and cumulatives.
6. **GP/PCA proxy comparison** – compresses each 41-quarter production curve with PCA and trains Gaussian-process regressors to predict the principal components from the four parameters.
7. **History matching** – searches the parameter space with Latin hypercube sampling and refines promising candidates with differential evolution. The final objective emphasizes later history and ignores the first six quarters because of simulator start-up timing differences.
8. **Forecast ensemble and template output** – generates accepted parameter candidates, runs the LSTM through 80 future quarters, rebases forecast rates to the last observed rates, integrates the rates into cumulative production, and selects ten representative forecast cases.

## Main results

### Proxy-model evaluation

- The GP/PCA proxy reproduces the simulated cumulative curves accurately on the validation and test splits, with normalized RMSE values of approximately `0.0005–0.0007` and reported `R² = 1.0`.
- The notebook reports a normalized LSTM test MSE of `0.000177` for the loaded trained model.
- Free-running LSTM experiments show error accumulation and reduced parameter spread over long rollouts. The GP proxy is therefore used for history matching, while the LSTM supplies the sequential forecast shape.

### Final history match

The final matching objective uses the oil, gas, and water rates with weights of zero for the first six quarters and linearly increasing weights from 1 to 5 for the remaining history.

- Best proxy mismatch: approximately `0.0291`
- Best real simulated-case mismatch: approximately `0.0303`
- Best real simulated cases: `65`, `69`, `32`, `19`, and `2`
- One best proxy solution reported by the notebook: fault transmissibility `0.153`, porosity multiplier `1.240`, permeability multiplier `12.564`, and aquifer pore volume `108.586`

The parameter solution is not unique. Blind-test results show that different parameter combinations can produce very similar production curves, particularly for fault transmissibility and permeability.

### Forecast ensemble

The final ensemble procedure:

1. Draws 50,000 Latin hypercube parameter candidates.
2. Accepts 7,995 candidates whose mismatch is within 1.5 times the best mismatch.
3. Samples 100 accepted candidates for forecasting.
4. Runs the LSTM for the 40 historical quarters plus 80 future quarters.
5. Applies median rate correction factors of `0.886` for oil, `0.883` for gas, and `1.086` for water so the forecast begins at the final observed rates.
6. Selects ten members from the 30 best history fits, spread across the 2028 cumulative-oil range.

For the full accepted ensemble, 2028 cumulative oil is approximately:

- P10: `62.1 million STB`
- P50: `62.9 million STB`
- P90: `64.0 million STB`

The ten selected cases cover approximately `62.7–63.5 million STB` of cumulative oil at 1 January 2028. The notebook also checks that the generated cumulative curves are non-decreasing and contain all 80 quarterly forecast rows.

## Running the notebook

The notebook was developed for Google Colab and is not completely self-contained as a local script. Some cells expect Google Drive paths and a private GitHub repository containing intermediate clean tables. The trained model is supplied locally at `Dataset/models/reservoirLSTM_1.pth`, and the completed workbook is supplied at `Dataset/clean/Template_filled.xlsx`.

To reproduce the completed workflow in Colab:

1. Open `Dataworks.ipynb` in Google Colab.
2. Mount Google Drive if running the data-preparation cells. The notebook expects the original workbook source files under the `dw26/xlsx` project location.
3. Create a Colab secret named `GITHUB_TOKEN` with access to the private data repository used by the notebook.
4. Run the dependency-installation and data-loading cells before the modeling cells. The notebook uses `pandas`, `numpy`, `scikit-learn`, `scipy`, `torch`, `matplotlib`, `openpyxl`, `pyarrow`, and `python-calamine`.
5. Run the notebook from the data checks through the final forecast cells. If the clean tables and trained model have already been prepared remotely, the later modeling sections can be run without repeating the original workbook conversion.

When executed, the notebook may create intermediate files such as `history_match_top20.parquet`, `forecast_ensemble.parquet`, and `best10_params.parquet` in the Colab runtime. The final trained model and completed forecast workbook are already supplied in `Dataset`.

## Limitations

- Observed cumulative production is derived from observed rates and is therefore approximate.
- History matching is non-unique: a good production match does not imply that every recovered parameter is physically identifiable.
- The forecast level depends on the rebasing correction applied at the 2008 history boundary.
- No observed production is available after 2008 for validating the 2008–2028 forecast.
- The forecast ensemble mainly represents uncertainty in the four matched parameters while sharing one learned LSTM production shape. It may understate uncertainty, especially if the true field declines faster than the learned forecast.
- The notebook depends on the Colab, Google Drive, and private GitHub setup used during development; paths and authentication must be adapted for another environment.
