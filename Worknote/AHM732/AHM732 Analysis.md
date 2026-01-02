Link ADO: [Change 143121 Add support for AHM732](https://dev.azure.com/airhart/CPH/_workitems/edit/143121)

OCT, 2024 : AODB already support in [[Change 131996]] , documented at [Assign Delay Code](https://dapproduct.atlassian.net/wiki/spaces/PD/pages/69010760/Assign+Delay+Code)

What has been done: 
- AODB support for AHM732 format done in  [[Change 131996]]
- MD already has fields ![[Pasted image 20251119052900.png]]
- ![[Pasted image 20251119053104.png]]

before 2.9, handle 3 character format (XXY) -> Delay code XX, Subcode Y
after: XX Y

Refinements (Finn Andresen): 
- MD: Three AHM732 Delay Codes entities to be added - one table for 'Process Code', one for 'Reason Code' and one for Stakeholder Code' ==already done
- Entries for each AHM732 Delay Code table to be created with descriptive texts ==??
- Existing Delay Code, Delay Sub-codes and Airline specific delay codes should be maintained for handling airlines not yet switched to AHM732

- UI: Tooltip on delay code should make a lookup ==character-by-character and concatenate the descriptive texts== (as it is the case for existing delay code and sub-code) 
- The lookup needs to be aware of whether the delay is reported as AHM730/731 codes or AHM732 codes

- Outbound: If we can only handle 2-character irregularity codes, I suggest that we map the two first character of the AMH732 delay code and ignore the 3rd character (Stakeholder code). Alternatively, building a large (25x25x25) translation table from AHM732 code to AHM730 code ==? is this supported ?==

Final conclusion (Jens):

We will build the support for this in:
- Master Data: ==already done ?
- Separate attributes in the AODB (flight leg): ==what attribute ?
- ==Complex update== action that allows the user to update the flight leg attribute (get inspiration from [https://iata-ahm732.azurewebsites.net/#/guided](https://iata-ahm732.azurewebsites.net/#/guided))
- ==No inbound (no messaging contract update)
- For ==outbound integrations==, we need to ==investigate which are using delay codes, and how they should be updated.
- ==Consider combinations that don't make sense==, alert ? (alert type ?)


### Work:
#### Masterdata:

- in delay code table, already has Ahm732 fields, requires every combinations of ahm72 
	- How to get unique delay code and description ?