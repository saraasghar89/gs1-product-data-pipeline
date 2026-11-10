# GS1 Product Data Transformation Pipeline

A Python pipeline that turns a messy supplier product spreadsheet into **GDSN-style XML**, the standard format retailers use to receive product master data. Products that fail validation are held back and listed in an error report for the supplier to fix.

Built with **pandas** and **xml.etree.ElementTree**.

## The problem

Suppliers usually send product data as Excel files filled in by hand. The same unit might appear as `5 kg`, `1KG` or `800 gm`. Dimensions come in centimetres with no stated order, and barcodes can have typos, hidden spaces or duplicates. Retailers and GDSN data pools need standard codes, consistent units and valid GTINs before they can accept anything.

## What the pipeline does

| Step | Example |
|---|---|
| Cleans and validates GTINs (GS1 mod-10 check digit, duplicates) | `8961234500036` → invalid check digit, flagged |
| Maps free-text sizes to GS1 / UN-ECE unit codes | `"300ML"` → `300` + `MLT` |
| Converts weights to grams | `"0.42 kg"` → `420` |
| Converts dimensions to millimetres | `"25 x 8 x 6"` cm → `250 / 80 / 60` mm |
| Splits descriptions into brand and functional name | `"SAHIL MASALA CHIPS 50G"` → `Sahil` / `Masala Chips` |
| Maps countries to ISO 3166 numeric codes | `"PK"`, `"Pakistan"` → `586` |
| Builds unit → case hierarchies | Case `18961234500041` contains 12 × unit `8961234500044` |
| Exports clean products to XML, everything else to an error report | 3 products exported, 9 rows held back |

## Validation rules

- Unit and case GTIN check digits must be valid
- A unit or case GTIN may appear on only one row
- Product description is mandatory
- Case weight and case dimensions are mandatory, and a weight with no unit is treated as missing
- Unit gross weight must not be less than its net content

Problems found in the sample file include an invalid barcode, a case barcode reused across two products, a duplicate unit barcode, missing case data, a 250 ml product listed at 200 g, and a case weight with no unit.

## Files

| File | Description |
|---|---|
| `XML_Convert.ipynb` | The pipeline, step by step, with outputs |
| `Sahil_Foods_Client_Product_Data.xlsx` | Sample raw supplier file, plus reference sheets for unit codes, country codes and target fields |
| `sahil_foods_gdsn.xml` | Output: clean products as GDSN-style XML |
| `validation_report.xlsx` | Output: every issue found, by Excel row and GTIN |

## Sample output

```xml
<tradeItem>
  <gtin>18961234500041</gtin>
  <tradeItemUnitDescriptorCode>CASE</tradeItemUnitDescriptorCode>
  <isTradeItemAConsumerUnit>false</isTradeItemAConsumerUnit>
  ...
  <nextLowerLevelTradeItemInformation>
    <childTradeItem>
      <gtin>8961234500044</gtin>
      <quantityOfNextLowerLevelTradeItem>12</quantityOfNextLowerLevelTradeItem>
    </childTradeItem>
  </nextLowerLevelTradeItemInformation>
  <height measurementUnitCode="MMT">250</height>
  <width measurementUnitCode="MMT">190</width>
  <depth measurementUnitCode="MMT">190</depth>
  <grossWeight measurementUnitCode="GRM">5300</grossWeight>
</tradeItem>
```

## How to run

1. Open `XML_Convert.ipynb` in Google Colab or Jupyter.
2. Upload `Sahil_Foods_Client_Product_Data.xlsx` to the same folder.
3. Choose **Runtime → Run all** (or run the cells top to bottom).

Requirements: Python 3.9+, `pandas`, `openpyxl`, `lxml` (only needed for the round-trip check).

## Notes and limitations

- The XML structure is simplified. Real GS1 GDSN messages wrap these attributes in more nested modules and are checked against the official GS1 XML schemas.
- All data is fictional. GTINs use the GS1 Pakistan prefix range and have correct check digits, but they aren't registered products.
- Possible next steps: XSD schema validation with `lxml`, GPC category mapping, and pallet-level hierarchies.
