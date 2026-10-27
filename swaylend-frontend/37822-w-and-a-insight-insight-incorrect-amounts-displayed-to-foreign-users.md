# #37822 \[W\&A-Insight] insight incorrect amounts displayed to foreign users

## #37822 \[W\&A-Insight] Incorrect Amounts Displayed To Foreign Users

**Submitted on Dec 16th 2024 at 18:39:10 UTC by @Blockian for** [**IOP | SwayLend Frontend**](https://immunefi.com/audit-competition/swaylend-frontend-iop)

* **Report ID:** #37822
* **Report Type:** Websites and Applications
* **Report severity:** Insight
* **Target:** <https://github.com/Swaylend/swaylend-monorepo/tree/develop/apps/frontend>
* **Impacts:**
  * Temporarily disabling user to access target site, such as:
* Locking up the victim from login
* Cookie bombing, etc.
  * Displaying Incorrect Information To Users

### Description

## Sway Bug Report

### Incorrect Amounts Displayed When Page Is Translated

#### Overview

An issue has been found in the Swaylend frontend where currency symbols are incorrectly displayed when the page is translated into different languages.

#### Detailed Findings

In all areas of the frontend where amounts are displayed, the `$` symbol is hardcoded. However, the wrapping HTML element does not include the `notranslate` class. This oversight causes Google Translate to replace the `$` symbol with a currency sign that matches the selected language, leading to confusion.

#### Example

For instance, if a European user visits the site (a highly likely scenario) and uses Google Translate to view the site in Dutch, all instances of the `$` symbol will be converted into `€` symbols. The underlying numeric values remain unchanged, which misrepresents the actual currency.

#### Impact

This discrepancy creates a misleading user experience by displaying incorrect amounts. Users may make incorrect assumptions about the values presented, potentially leading to erroneous transactions and a loss of trust in the system.

#### Proposed Solution

To resolve this issue, include the `notranslate` class (as recommended in the [Google Translate documentation](https://cloud.google.com/translate/troubleshooting)) for all elements displaying currency amounts. This will prevent Google Translate from altering the currency symbols.

### Proof of Concept

#### Proof of Concept

To replicate the issue, follow these steps:

1. Navigate to <https://app.swaylend.com/>.
2. Enable Google Translate and select a language (e.g., Dutch).
3. Observe that all `$` symbols are replaced with `€` symbols while the numeric values remain unchanged.

#### Summary

Adding the `notranslate` class is a simple and effective fix to ensure currency symbols remain accurate across all translated versions of the site, preserving user trust and preventing potential errors.
