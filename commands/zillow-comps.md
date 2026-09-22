---
description: Comparable sales for an address or an area, pulled from Zillow
---

Find recent comparable sales and summarise what they say about price.

Ask me for the area and the shape of the property if I have not already given them. The area is whatever Zillow accepts: a city, a ZIP code or a neighbourhood.

Then:

1. Call `hasdata_zillow_listing_getRealEstateListings` with `type: "sold"`, the area in `keyword`, and the bed and bath minimums that match the subject property. Add `squareFeet_min_` and `squareFeet_max_` at roughly plus or minus twenty percent of the subject when I gave me a size.
2. Keep the sales from the last six months. Drop anything whose `livingArea` is more than a third away from the subject.
3. Report the median price and the median price per square foot, then list the five closest comps with address, sold price, date, beds, baths and size.
4. Say plainly how many sales the numbers rest on. Three comps and thirty comps are not the same answer.

If the area returns nothing, widen it once and say that you did. Do not silently switch to a different city.
