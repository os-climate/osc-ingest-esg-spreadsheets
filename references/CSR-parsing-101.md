# CSR Spreadsheet Parsing 101

When rendered as a spreadsheet, the structure of CSR reports seem to follow a somewhat predictable structure:

## Basic Elements

* **Topic**: Broad categories tied to keywords, such as:
    * GHG Emissions
    * Other Emissions (pollution)
    * Energy
    * Water
    * Waste Management
    * Biodiversity
* **Sub-topic**: elaborate on topic keywords, but do not define Units
* **Category**: elaborate on topic keywords, and *do* define Units
    * Often roll up Totals, expressed in Units
    * Often expressed across multiple rows (Units in one row, Label/Totals in another)
* **Sub-Category**: summarize sub-totals that roll up to Categories (in *ad hoc* ways)
* **Segmentation**: a particular way of slicing a category, *e.g.* by country, business unit, *etc.*  Note that segmentations are always named, making each such segmentation easily analyzed, whereas sub-categories are not named.
* **Sub-segment**: arbitrary levels of indentation (or other syntactic sugar) indicate that the present row can be prefixed by higher-level segmentations to allow multi-dimensional segmentation.  For example:

        Scope 1 Emissions by Source (Category and Segmentation)
            CO2 emissions total (CO2 emissions)
                Combustion (CO2 emissions::Combustion)
                Flaring (CO2 emissions::Flaring)
                Venting and process (CO2 emissions::Venting and process)
                Fugitives (CO2 emissions::Fugitives)
            CH4 emissions total (CH4 emissions)
                Combustion (CH4 emissions::Combustion)
                Flaring (CH4 emissions::Flaring)
                Venting and process (CH4 emissions::Venting and process)
                Fugitives (CH4 emissions::Fugitives)
            Other GHG

* **Variable**: The actual observed quantity corresponding to a measurement date (typically a year).  Due to different locales using different numbering systems, and due to reports sometimes prepared within different locales and merged together without regard for re-rendering in the production locale, care is needed when interpreting these numbers (or number-like things, such as "<1", "n/a", "n/c", "n/d", "/", etc.)

Additionally, many spreadsheets have Comments, Notes, and other metadata, such as references to SASB, GRI, TCFD, URD 2020 (Amundi) labeling and conventions.

Some reports put all information on a single worksheet.  Some separate the worksheets by Topic or Sub-Topic.  But regardless of how the structures are formatted and presented, it is important to remember that one of our over-arching goals is **_to collect data and its most granular level_**.  To do that we need to collect the data as found and then transform it later.  The PUDL people refer to this as EtLT (see: https://github.com/catalyst-cooperative/pudl/issues/1315; as opposed to ETL, which envisions a heavyweight transformation step between Extract and Load).

## Contextual Parsing

Topics are tied to a small set of keywords.  Any new row which mentions a topic keyword could be the definition of a new Category.  Remember that Topic or Sub-Topic information could come from the title of a worksheet, not just from processing a row.

Variables are often described in a way that links back up to higher-level classifications, sometimes in multipe ways.  For example, Variables may be described as:

        Scope 1 emissions breakdown
            CO2 (Metric Tons)
            SO2 (Lbs)
            SO2 (MT)
            NOx (Lbs)
            NOx (MT)
            Mercury (Lbs)
            Mercury (kg)

In this case, each Variable is defining its own Units.  And because all the Units are different, they don't really roll up to the header to make it a Category, but rather a Sub-Topic.  Sub-topics can be trivially created from Categories by simply dropping the Units.  One can then choose to browse by more general subject areas (Sub-topics) or more specific, quantified, subject areas (Categories).

In other cases, variables may link explicitly to Topics and Categories:

        Direct GHG emissions - Total (With subcontractors)
        Direct GHG emissions - Waste - Total (With subcontractors)
        Direct GHG emissions - Waste - Collection activities
        Direct GHG emissions - Waste - Incineration (including hazardous waste)
        Direct GHG emissions - Waste - Landfills (Hazardous and non hazardous)
        Direct GHG emissions - Waste - Treatment of hazardous waste (excluding incineration)
        Direct GHG emissions - Waste - Sorting activity
        Direct GHG emissions - Waste - Subcontractors' fuel emissions
        Direct GHG emissions - Waste - Other emissions (Primary energies, excluding treatment)

**Thus, one of the major tasks of parsing these spreadsheets is to collect the contextual information that comes from top-down (Topics->Categories) and connect it with the Variable information to produce consistent, categorizable, unitized datapoints.**  

## Processing Phases

There are sevreal processing phases from the first reading of a CSR to the final distribution of data in the Data Commons.  The phases begin with the identification of a report in a semi-structured format (XLS, XLSX, and conceivably CSV).

* Download the report to local storage (**data/external**)
* If necessary, convert report to XLSX (scripts do not handle XLS or CSV directly)
* QC report (many reports have errors, such as wrong or ill-defined units, wrong footnote references, wrong background fill colors, invalid or incorrect numerical formats, etc.).  Iteratively fix report or fix processing scripts until QC passes.  QC'd reports go to **data/raw** *n.b.* By default we do not store reports in data/* in GitHub, so QC work done on XLSX file instead of script is not saved unless **.gitkeep** overrides **.gitignore**.
* Process as-found data (to **data/interim**)
* Link to Data Commons (to **data/processed**)
* Publish data in Trino (**schema = osc_corp_data**)

### QC Phase

It is complicated enough to write a script to process a wide range of ways to all express roughly the same thing.  It's impossible when a report is inconsistent with its own rules.  If the processing script is failing because a rule has not been properly defined, and if the rule is relatively simple and general, the script should be enhanced to implement the rule.  But CSR reports are compiled by people, and people can make mistakes.  It is far preferable to correct mistakes at the source than to try to create complex rules that adapt to errors but do not also upset other, adjacent rules.  Consider these numbers from a report:

* 2017	0,378 TJ / thousand tons of iron ore equivalent
* 2018	0,352 TJ / thousand tons of iron ore equivalent
* 2017	4.183 TJ of energy saved/avoided as a result of conservation or efficiency improvement initiatives.
* 2018	256,7 TJ of energy saved/avoided as a result of conservation or efficiency improvement initiatives.* 
* **2017	1,423 million m³ of water recycled and reused (82%)**
* 2018	953 million m³ of water recycled and reused (83%)
* 2017	"In 2017, our total operational areas were distributed as follows:
    Total impacted area: 1,504.33 km²
    Total impacted area in Wilderness: 910.96 km²
    Total area impacted in Hotspots: 413.04 km²
    Impacted areas in protected areas: 195 km²
    Impacted areas adjacent to protected areas: 468.7 km²
    Impacted areas in priority areas for conservation outside protected areas: 126.7 km²
    Impacted areas adjacent to priority areas for conservation outside protected areas: 190.5 km²"
* 2019	"In 2019, there were:
    - 54 operational units analyzed;
    - 46 (85.2%) of the areas require Biodiversity Management Plan;
    - 49 in total have already been implemented (including areas with more than one plan);
    - In only one unit, the required plan is still to be implemented."
* 2020	"In 2020, there were:
    - 61 operational units analyzed;
    - **51 (83.6.2%) of the areas require a Biodiversity Management Plan;**
    - 58 in total have already been implemented (including areas with more than one plan);
    - In only one unit, the required plan is yet to be implemented."

The first bolded line shows an ambiguity between using en_US numbering (above) and en_DK numbering (below).  The second bolded line shows an entirely invalid numerical format for describing a percentage.

Anecdotally, the error rate in these reports is one or more obvious errors in every 2-3 reports.  Most such errors are correctable with minimal human intervention.

### As-found Processing Phase

For the as-found phase, the goal is to capture the data in a unified, self-consistent fashion.  Alas, we cannot do this with zero knowledge; we need to begin with a minimal set of dictionaries to enable linkage between topics, categories, variables, and units.  In this phase we are not attempting to strictly normalize reports against any particular standard, but rather to harmonize the report with itself.  Thus, if we find a row that mentions "Emissions" or "Energy" we can know the topic.  If we find a row that mentions "Scope 1", "Scope 2", or "Scope 3", we can name a category, regardless of how the text surrounding those terms reads.  Ditto for "consumed", "produced", "incinerated", "landfilled", composted", recycled", "reused", etc.

Similarly, we want to normalize numbers and units to *a* set of standards.  So we convert non-standard terms like "MM&dollar;" or "billions of gallons" to SI units (and their financial equivalents).

We also collect notes annotations (where we can--openpyxl does not handle superscript numbers at all well), as well as other column data that's destined to be metadata (such as comments, linkages to SASB, GRI, etc.)

This phase is complete when the variables map to the topics/categories and we can produce a single Tidy dataframe of the form: Topic:Category:Segmentation:Variable:Unit:Value:Year:[Notes and Other Metadata Columns].

### Linkage to Data Commons

Once we have a clean version of "As-found" data, we need to map that to more regular factors that make it searchable in the Data Commons.  This is where we regularlize the many ways of describing Scope 1, Scope 2, and Scope 3 emissions, as well as their various ways of combining/splitting their sub-components.  This is also the step where linkage to other data frameworks (such as SASB, GRI, etc) are made.

We can also do some other data linkage: Entity matching (to LEI), Industry and Sector classification, Country/Region assignment, etc.

In this step we also convert notes and other columnar data to metadata for loading into the Trino store.

### Publishing Data to Trino

This is the simplest step: just load linked data into Trino and ensure the Data Commons Browser can find it as one of many tables that ultimately feed into the overall Data Commons Corporate Data store.
