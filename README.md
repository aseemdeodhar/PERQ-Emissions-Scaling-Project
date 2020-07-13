# PERQ-Emissions-Scaling-Project

### Introduction

The Perq program, is a rebranding of the MBTA's Corporate Pass Program. Through Perq, employers can offer pretax or subsidized monthly passes to their employees. Perq is a way for employers to incentivize their employees to replace vehicle trips with transit for their daily commute.

As part of the Emissions Scaling Project, we measure the pounds of CO<sub>2</sub> saved by employees by using public transit in place of single-occupancy personal vehicles. We measure these numbers by utilising the Origin-Destination data gathered through the Perq CharlieCards for a given month. For this example, we will be using data from September 2018.

Travel modes were inferred through the dataset for bus and subway. For Commuter Rail, as there is no automated fare collection, we do not have Origin-Destination data. We therefore estimate by averaging the distances from stations in each Commuter Rail Zone to their respective Downtown Station (North or South Station).

For a detailed description and the blog entry please see the [mbtabackontrack](<https://www.mbtabackontrack.com/blog/104-perq-pass-carbon-emissions-savings>) blog.

We calculate distances and time taken for a given Origin-Destination pair. This helps us calculate the pounds of CO<sub>2</sub> emitted by each journey. These values are then aggregated by company and the mode of travel. Thus, the final output is anonymised, and all personally identifiable information is stripped.

The front-end product was developed by Yinan Dong, a colleague and co-intern, and can be seen here.

**Please Note: Names of certain tables, schemas and companies have been changed for proprietary and privacy reasons. Any aggregations showns in this documentation are kept to 10 units or more for privacy protection.**
