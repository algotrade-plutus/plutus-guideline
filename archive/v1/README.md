# The PLUTUS Standard Guideline

# 0 Introduction
The PLUTUS Standard aims to ensure the `Reproducibility` of the project up to `Step 7: Paper Trading` of the [9-step Process](https://hub.algotrade.vn/knowledge-hub/steps-to-develop-a-trading-algorithm/). By `Reproducibility`, we mean:
- The project should have source code to reproduce all the reported results.
- The user should be able to run the source code independently without directly communicating with the author.
- All the required materials to run the source code should be self-contained in the project's `README.md` file. The resource is often:
    - The instructions on how to get the required data.
    - The instructions on how to run the source code.
- All the reported results should also be self-contained in the `README.md` file. Moreover, the reproduced results should match the reported ones in the `README.md` file. There are often three to four sections needed to report the result:
    - `Step 4: In-sample Backtesting`
    - `Step 5: Optimization`
    - `Step 6: Out-sample Backtesting`
    - `Step 7: Paper Trading` (optional)

Hence, the project should:
- Be hosted on [ALGOTRADE Plutus organization in Github](https://github.com/algotrade-research).
- Have a `README.md` file that satisfies the requirements (next section).
- And a Final Report or Paper (optional).

The requirements for the `README.md` file are presented in the section below.

# 1 The `README.md` Requirements
Each repository should have a `README.md` file containing the project's overall information. The file is put in the project's root folder and refers to the repository [`plutus-project-template`](https://github.com/algotrade-research/plutus-project-template) for the template of each section.

The idea of `README.md` is to provide the reader of the repository with how to **RUN** and **REPLICATE** the result(s) of this project. Hence, these sections are the focuses of this file:
- `Data`: can be brief, but understandable and clear how to get the data to replicate the results
- `Implementation`: The most important section shows how to run and replicate the project
- `In-sample Backtesting` to `Paper Trading`: The second most important section shows how to develop the algorithms and the associated results.

The structure of the specific `README.md` file can deviate from the template presented below, depending on the situation. Nevertheless, the file should give the reader clear instructions for replicating results and an accurate project overview.

Below is the template of the `README.md`:
- Has an `Abstract` to summarize the project, the motivation, the methods, and the findings in less than five sentences.
- Has an `Introduction` section to talk about the project briefly. The content of this section can be:
    - The motivation to do the project (answering the "Why?" question)
    - An overview of the method conducted in this project (answering the "How?" question)
    - The goal of the project (answering the "What?" question)
    - This section in the `README.md` file should be brief. Only 1-2 paragraphs are sufficient. Details can be included in the final report/paper.
- Has a (optional) `Related Work` or `Background` section to briefly introduce the prerequisite reading if the audience needs knowledge before exploring the project.
- Has at least sections to briefly discuss from `Step 1: Forming Trading Hypotheses` to `Step 6: Out-of-sample Backtesting` (or `Step 7: Paper Trading`) of the [9-Step process](https://hub.algotrade.vn/knowledge-hub/steps-to-develop-a-trading-algorithm/):
    - A section to discuss the `Trading (Algorithm) Hypotheses` (Step 1).
    - A section to discuss the `Data` (Step 2: Data Collection and Step 3: Data Processing)
    - A section to present the `Implementation` (or `How to Run`) of the trading algorithm
    - A section to discuss the `In-sample Backtesting` (Step 4)
    - A section to discuss the `Optimization` (Step 5)
    - A section to discuss the `Out-of-sample Backtesting` (Step 6) 
    - A section to discuss `Paper Trading` (Step 7), if there is
- The `Data` section should have two subsections:
    - `Data Collection` to discuss the Step 2
    - `Data Processing` to discuss the Step 3
    - Some questions need to be answered in this section:
        - What is the data? What is the period of data, format, structure, and type?
        - How to input the configuration?
        - How to store the output data?
- The `Implementation` or (`How to run`) should present:
    - How to set up the environment to run the source code and the required steps to replicate the results
    - Discuss the concrete implementation if there are any essential details
    - How to run each step from `In-sample Backtesting`, Step 4 to `Out-of-sample Backtesting`, Step 6 (or `Paper Trading`, Step 7).
    - How to change the algorithm configurations for different runs.
- In each section from the `In-sample Backtesting` to the `Paper Trading`:
    - Specify the standard (or sample) set of parameters, input data, and the corresponding output.
    - There should be a subsection `Result` to present and discuss the results.
- In this `README.md`, the `Result` subsection of each step:
    - Should only briefly show the output results, such as tables and images. The link to the report for each step (4-7) should also be presented.
- The complete set of parameters, input data, and output of all the experiments examined in the project should be mentioned in a separate section but not in the README.md file. This information is preferred in the **Final Report** or **Paper**.
- Has a (optional) `Conclusion` section.
- Has a `Reference` section to mention the references.
- A link to the Final Report/Paper
    - Show how to compile the document if it is written in a language like LaTex.
- Be reasonably easy to follow.

## 1.1 The Final Report/Paper (Optional)
- Apart from the `README.md` file, there can be a formal written report or paper further to discuss the methodology and results of the project.
- This document contains all the same sections as the `README.md` file but in more detail. Note that the `Related Work` (or `Backrgound`) and `Conclusion` sections are **NOT** optional in the final report.
- If the final document was written in software such as LaTex, it should be included in the project's GitHub repository and the source to compile it.
- The final document should contain adequate information to help the reader understand the project. It should contain documentation reproducing the result up to:
    - Minimally, Step 6: Out-of-sample Backtesting
    - Ideally, Step 7: Paper Trading
    
    of the [9-Step process](https://hub.algotrade.vn/knowledge-hub/steps-to-develop-a-trading-algorithm/).

## 1.2 Other aspects
Several aspects are available to assess the projects. However, these aspects are not currently required to comply with the PLUTUS Standard. The non-exhausted list of aspects is presented [here](standard/plutus-assessment-guide.md).

# 2 Data
Data source and data information can be found [here](./data/DATA.md).

# 3 Sample Projects
Three badge levels go with the Compliance Score:
- ![Static Badge](https://img.shields.io/badge/PLUTUS-<score>-%23BA8E23) is for a project's score ranging from 50% to 69%. 
- ![Static Badge](https://img.shields.io/badge/PLUTUS-<score>-darkgreen) is for a project's score ranging from 70% to 99%.
- ![Static Badge](https://img.shields.io/badge/PLUTUS-100%25-purple) is reserved for projects with a score of 100%, flagship projects to promote the PLUTUS Guidelines.

Furthermore, the
- ![Static Badge](https://img.shields.io/badge/PLUTUS-Sample-darkblue) is given to projects that are selected to become PLUTUS Sample Project, featured in [PLUTUS Standard Guideline](https://github.com/algotrade-course/plutus-guideline/)
- ![Static Badge](https://img.shields.io/badge/PLUTUS-PROTO-%23880A88) is given to projects in the `PROTO` algorithm series developed by ALGOTRADE.

Below is the list of sample projects which adhere to the PLUTUS Standard. Each project has link to the author/maintainer profile. Plutus compliance percentage show how much the project is currently aligned with the PLUTUS Standard.
| **ID** | **Project Name** | **Strategy Type** | **Author/Maintainer** | **Program** | **PLUTUS Badge** |
|--------|------------------|-------------------|-----------------------|:-----------:|:---------------------:|
| 1 | [PROTO:SmartBeta](https://github.com/algotrade-research/ProtoSmartBeta) | [Smart-Beta](https://hub.algotrade.vn/knowledge-hub/smart-beta-strategies/) | [Tạ Quang Khôi](https://github.com/khoi-ta), [Nguyễn An Dân](https://github.com/dan-algo) | ALGOTRADE PROTO | ![Static Badge](https://img.shields.io/badge/PLUTUS-75%25-darkgreen)![Static Badge](https://img.shields.io/badge/PLUTUS-PROTO-%23880A88) |
| 2 | [PROTO:Market Maker](https://github.com/algotrade-research/ProtoMarketMaker) | [Market-Making](https://hub.algotrade.vn/knowledge-hub/market-making-strategy/) | [Tạ Quang Khôi](https://github.com/khoi-ta), [Nguyễn An Dân](https://github.com/dan-algo) | ALGOTRADE PROTO | ![Static Badge](https://img.shields.io/badge/PLUTUS-75%25-darkgreen)![Static Badge](https://img.shields.io/badge/PLUTUS-PROTO-%23880A88) |
| 3 | [InstiFund](https://github.com/algotrade-research/InstiFund) | [Smart-Beta](https://hub.algotrade.vn/knowledge-hub/smart-beta-strategies/), [ETF Front-Runner Strategy](https://hub.algotrade.vn/knowledge-hub/front-running-etf-strategy/) | [Đặng Minh Nhựt](https://github.com/BJMinhNhut) | ALGOTRADE Internship 11.24 | ![Static Badge](https://img.shields.io/badge/PLUTUS-80%25-darkgreen) |
| 4 | [DynamicGrid](https://github.com/algotrade-course/DynamicGrid) | [Grid](https://hub.algotrade.vn/knowledge-hub/grid-strategy/) | [Võ Hoàng Phúc Khang](https://github.com/vokhang1412), [Nguyễn Thanh Thảo Ly](https://github.com/sxweetlollipop2912), [Nguyễn Tuấn Khanh](https://github.com/ng-tuan-khanh) | CS408 - APCS, HCMUS - 2025 | ![Static Badge](https://img.shields.io/badge/PLUTUS-80%25-darkgreen) |
| 5 | [SearchingTA](https://github.com/algotrade-research/SearchingTA) | [Momentum](https://hub.algotrade.vn/knowledge-hub/momentum-strategy/), [Mean-Reversion](https://hub.algotrade.vn/knowledge-hub/mean-reversion-strategy/) | [Tống Thiên Phước](https://github.com/tphuoc04/) | ALGOTRADE Internship 6.24 |  ![Static Badge](https://img.shields.io/badge/PLUTUS-75%25-darkgreen) |
| 6 | [FinTrip](https://github.com/algotrade-research/FinTrip) | [Smart-Beta](https://hub.algotrade.vn/knowledge-hub/smart-beta-strategies/) | [Tạ Quang Khôi](https://github.com/khoi-ta) | ALGOTRADE Internship 11.23 | ![Static Badge](https://img.shields.io/badge/PLUTUS-75%25-darkgreen) |
| 7 | [Statisical-Arbitrage](https://github.com/algotrade-research/Statisical-Arbitrage) | [Market-Neutral: Statistical Arbitrage](https://hub.algotrade.vn/knowledge-hub/market-neutral-strategy/) | [Cao Quang Hiếu](https://github.com/HieuQCao) | ALGOTRADE Internship 11.24 | ![Static Badge](https://img.shields.io/badge/PLUTUS-75%25-darkgreen) |
| 8 | [TruTrend](https://github.com/algotrade-course/TruTrend) | [Momentum](https://hub.algotrade.vn/knowledge-hub/momentum-strategy/) | [Phạm Võ Quỳnh Như](https://github.com/pvqn), [Hồ Việt Bảo Long](https://github.com/lob23), [Nguyễn Phúc Bảo Uyên](https://github.com/Hollyuyen) | CS408 - APCS, HCMUS - 2025 | ![Static Badge](https://img.shields.io/badge/PLUTUS-70%25-darkgreen) |
| 9 | [Intraday-Momentum](https://github.com/algotrade-research/Intraday-Momentum) | [Momentum](https://hub.algotrade.vn/knowledge-hub/momentum-strategy/) | [Lâm Thành Duy](https://github.com/ltduy6) | ALGOTRADE Internship 11.24 | ![Static Badge](https://img.shields.io/badge/PLUTUS-75%25-darkgreen) |
| 10 | [DuoMA](https://github.com/algotrade-course/DuoMA) | [Momentum](https://hub.algotrade.vn/knowledge-hub/momentum-strategy/) | [Nguyễn An Dân](https://github.com/dan-algo) (Maintainer) | CS408 - APCS, HCMUS - 2025 | ![Static Badge](https://img.shields.io/badge/PLUTUS-75%25-darkgreen) |
| 11 | [QuarterOscillate](https://github.com/algotrade-course/QuarterOscillate) | [Mean-Reversion](https://hub.algotrade.vn/knowledge-hub/mean-reversion-strategy/) | [Đặng Đức Khiêm](https://github.com/duckhiemdang), [Nguyễn Lộc An](https://github.com/LucidNg), [Đặng Minh Triết](https://github.com/triet0612) | CS408 - APCS, HCMUS - 2025 | ![Static Badge](https://img.shields.io/badge/PLUTUS-70%25-darkgreen) |
| 12 | [MomMean](https://github.com/algotrade-course/MomMean) | [Mean-Reversion](https://hub.algotrade.vn/knowledge-hub/mean-reversion-strategy/), [Momentum](https://hub.algotrade.vn/knowledge-hub/momentum-strategy/) | [Nguyễn Đức Hưng](https://github.com/DucHungGithub), [Hoàng Thiên Đức](https://github.com/Locopaly), Hoàng Nghĩa Việt |   CS408 - APCS, HCMUS - 2025 | ![Static Badge](https://img.shields.io/badge/PLUTUS-75%25-darkgreen) |
| 13 | [SmartBeta](https://github.com/algotrade-research/SmartBeta) | [Smart-Beta](https://hub.algotrade.vn/knowledge-hub/smart-beta-strategies/) | [Lê Đức Phú](https://github.com/dphu2609) | ALGOTRADE Internship 11.24 | ![Static Badge](https://img.shields.io/badge/PLUTUS-70%25-darkgreen) |
| 14 | [scalping-strategy](https://github.com/algotrade-research/scalping-strategy) | [Scalping](https://hub.algotrade.vn/knowledge-hub/scalping-strategy/) | [Trần Thị Phương Linh](https://github.com/ttplinh) | ALGOTRADE Internship 11.24 | ![Static Badge](https://img.shields.io/badge/PLUTUS-70%25-darkgreen) |
| 15 | [DeepMM](https://github.com/algotrade-research/deepmm) | [Market-Making](https://hub.algotrade.vn/knowledge-hub/market-making-strategy/) | [Lê Thanh Danh](https://github.com/danhleth), [Lương Thanh Anh Đức](https://github.com/luongthanhanhduc)| ALGOTRADE Internship 11.23 | ![Static Badge](https://img.shields.io/badge/PLUTUS-50%25-%23BA8E23) |
| 16 | | | | | |


## 3.1 Other example projects
There is also a repertoire of projects that adhere to the PLUTUS Standard, which comes from the educational program offered by ALGOTRADE, that can be explored [here](https://github.com/algotrade-course).

Listed are the projects from:
- [CS408: Computational Finance, Algorithmic Trading - APCS, HCMUS, 2025](https://github.com/algotrade-course#cs408---apcs-hcmus---2025-project)
