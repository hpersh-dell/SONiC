# Enhancements to Optical Transceiver Information Model
<h4>Revisions</h4>
<table style="border-style:none;">
	<tr>
		<td valign="top">Version 0.0</td>
		<td>Initial draft by @hpersh</td>
	</tr>
	<tr>
		<td valign="top">Version 0.1</td>
		<td>Added new fields, explained parsing logic, and terminologies by @ade-d</td>
	</tr>
</table>

## Introduction
Currently, xcvrd parses the EEPROMs of optical media transceivers, and builds a Python data structure, a dictionary, containing transceiver information.

This design is an effort to enhance and standardize the representation of transceiver information. We deviate from the formerly taken approach of using the raw SFF and CMIS specs to parse info because the information generated using this method is susceptible to changes spec revisions, and it doesn't provide proper insight into what the data actually means.

We propose a system which abstracts away a lot of the underlying nuances between different module types, specifications and revisions. This enables a uniform method of interpreting and using media attributes.
For example, the concept of "cable length" can mean different things for various media because some come without any cable pre-attached. But it gets complicated because the EEPROM field for cable length can be dual purpose and as such can imply the maximum link length of whatever cable is pre-attached, or the maximum length of cable that can be used to ensure proper link behavior. Further more, the methods of reading cable length can vary from various classes of media and various revisions withing the same class. All this can get confusing, so we introduce a shim layer which provides the necessary uniformity and clarifications needed.
We start with a few essential fields which are necessary to identify the module unambiguously, and then we then proceed to use some of these fields to set attributes in the NPU which are required for proper media functionality.

The initial fields to be parsed and presented are listed in the Proposed Format section.


### Goals
The goal is to have xcvrd build a Python dictionary, containing all static transceiver information (i.e. transceiver information that does not change over time) for an installed transceiver, information that is needed to:
- properly configure the NPU for the installed transceiver, e.g. set the NPU interface type to "fiber" or "copper";
- provide information for making a platform-dependent, port-dependent and transceiver-dependent lookup of port configuration information to be passed to the NPU, e.g. look up port pre-emphasis value(s);
- provide information for determining possible port configurations, and validating requested port configurations, e.g. determining what, if any, breakout modes are possible on a port; and
- display transceiver information to the user.

The format of this dictionary, i.e. the member fields, and the contents of all those fields, shall be standardized, and platform-independent, so that the code to configure the NPU and display information to the user can also be platform-independent.

This dictionary, once created by xcvrd, is published to the Redis DB, as a record in the TRANSCEIVER_INFO table.  NPU-configuration and CLI code will obtain transceiver information from there.
### Current format
An example of the transceiver information dictionary constructed by xcvrd is as follows:
~~~
{
	"Connector": "No separable connector", 
	"cable_length": "2", 
	"cable_type": "Length Cable Assembly(m)", 
	"encoding": "64B66B", 
	"ext_identifier": "Power Class 1(1.5W max)", 
	"ext_rateselect_compliance": "QSFP+ Rate Select Version 1", 
	"hardwarerev": "A0", 
	"manufacturename": "DELL", 
	"modelname": "76V43", 
	"nominal_bit_rate": "255", 
	"serialnum": "CN0LXD0096H3A36", 
	"specification_compliance": "{'10/40G Ethernet Compliance Code': '40GBASE-CR4'}", 
	"type": "QSFP28 or later", 
	"type_abbrv_name": "QSFP28", 
	"vendor_date": "2019-06-17 ", 
	"vendor_oui": "3c-18-a0"
}
~~~

As can be seen from the above table, the `cable_length` field only supports integers. This loses precision for fractional lengths. We also see issues with `cable_type` which doesn't yield any valuable info.
The `ext_identifier` is also unclear because this field is used for multiple kinds of information with varying types and functionality, hence you cannot always rely on it to show the power rating. Instead of put the power info in the  `ext_identifier` field, we should put the power rating in a `power_rating` field.
Nowhere in this table are we truly aware of what kind of media is plugged in. The `specification_compliance` appears to provide a hint at this being a 40G CR4, but a 100G CR4 can have the same compliance code (since 100G CR4 can run at 40G by underclocking). Hence we see that the information generated is not really useful unless one understand the details of the media, which is rarely the case.

Another example from the formatted output of the current system:
~~~
        Connector: Unknown
        Encoding: Unknown
        Extended Identifier: Unknown
        Extended RateSelect Compliance: Unknown
        Identifier: QSFP-DD 8X
        FIBER: 1
        Nominal Bit Rate(100Mbs): 32
        Specification compliance:
                10/40G Ethernet Compliance Code: 10GBase-LRM
                Fibre Channel Speed: 1600 Mbytes/Sec
                Fibre Channel link length/Transmitter Technology: Intermediate distance (I)
                Fibre Channel transmission media: Miniature Coax (MI)
                SAS/SATA compliance codes: SAS 6.0G
        Vendor Date Code(YYYY-MM-DD Lot): 20
        Vendor Name: AFBR-91BBDDZ
        Vendor OUI: EC-01-E2
        Vendor PN: 1950B0007     20
        Vendor Rev: 19
        Vendor SN: ▒0
        connector_type: MPOx12
        vendor_name: FIT
~~~
Multiple fields are clearly not parsed correctly, and those that are, such as `Nominal Bit Rate` are simply nice to know, but not really useful.

Below is a more descriptive set of attributes as generated by our parser (note this is unformatted, so units are missing): 
~~~
vendor_serial_number           : F413QDQP90200099              
vendor_revision                : 16                            
vendor_name                    : Credo                         
display_name                   : QSFP28 100GBASE-SR4-ACC-3.0M  
vendor_oui                     : 9A-AD-CA                      
vendor_part_number             : CAC43X301D4P-DE1              
vendor_date_code               : 2019-05-01                    
cable_type                     : ACC                           
special_fields                 : None                          
power_rating_max               : 3.5                           
cable_breakout                 : 1x1                           
lane_count                     : 4                             
connector_type                 : No separable connector        
qsa_adapter                    : N/A                           
form_factor                    : QSFP28                        
media_interface                : SR                            
cable_length_detailed          : 3.0 
~~~
*We will be adding more fields in future for diagnostics, laser info, CDR, etc*

*Our method does not deprecate the old fields, but simply adds newer and more reliable fields. There will be information duplication as a result of this.*

## Transceiver information
### Parsing
Xcvrd shall parse optical media transceiver EEPROMs, by means of core, platform-independent code, and also using an optional plugin provided by a platform implementation, to handle possibly vendor-proprietary information encoded in the EEPROM.

The architecture of the xcvrd code is beyond the scope this document; that is the HOW of obtaining transceiver information, this document concentrates on the WHAT (syntax and semantics) of transceiver information.

This design is intended to standardize the minimum set of transceiver information; platform-dependent plug-ins may provide additional information, but all fields in this document must be present for all platforms.
The form factors of interest include the following (Ethernet modes only), as well as how much parsing is implemented relative to the base attributes (see `ext_media_handler_base.py`):
- SFP
- SFP+
- SFP28
- SFP56-DD (partial support due to spec not yet finalized)
- QSFP+
- QSFP28
- QSFP28-DD (partial support due to low demand and inconsistencies)
- QSFP56-DD

|    Form Factor            |Description	                          |Status|
|----------------|-------------------------------|-----------------------------|
|SFP			|Typically speeds of 1G            |Up to date            |
|SFP+          |10G evolution of 1G            |Up to date            |
|SFP28          |25G|Up to date|
|SFP56-DD          |Upcoming 100G |Partial support due to spec not yet finalized|
|QSFP+          |40G (4 lanes of SFP+)|Up to date|
|QSFP28          |100G (4 lanes of SFP28)|Up to date|
|QSFP28-DD          |200G (double density form of QSFP28)|Partial support due to low demand and inconsistencies|
|QSFP56-DD          |400G (quad and double density form of 50G signaling)|Up to date|

Due to the fragmented and inconsistencies in the properties of these media, we will be defining some standard fields which every form factor must provide info about, or yield a default of 'N/A'.
Underneath there are two kinds of attributes:
1. Static attributes
2. Dynamic attributes

##### 1. Static Attributes
These attributes are consistent, regardless of the switch-state, switch type, port or any other circumstances.
These include (but are not limited to): cable length, media type, vendor name, etc
The parsing logic for these are entirely based on the media EEPROM, unless an override is provided to state otherwise (more on overrides later).

##### 2. Dynamic Attributes 
These attributes may or may not be invariant. For example, QSA adapter depends on the form-factor and which port the media is plugged into.


#### Parsing Methodologies
Since static attribute parsing depends entirely on the media EEPROM content, and dynamic parsing depends on statically parsed fields, we defer the parsing of dynamic attributes until all static attrs are read.

Even though multiple form-factors share different addresses, and parsing mechanism, we define one file per form-factor, which enables us to easily add features without breaking support for other form-factors, and to quickly triage issues.
This is much easier approach than to implement parsing based on the various SFF and CMIS specs.

At the end of parsing, a dictionary of <key, value> pairs is returned, where key is a string of attr name, and key is a string of the corresponding value for that attr.

Every file in this project is prefixed `ext_media_`

The main files of interest are:
<table>
	<tr>
		<td valign="top">ext_media_handler_base.py </td>
		<td>Provides a purely abstract class with methods which each form-factor handler file must conform to. </td>
	</tr>
	<tr>
		<td valign="top">ext_media_handler_*.py </td>
		<td>
		Here * represents the form-factor codified name (SFP is sfp, QSFP+ is qsfp_plus, QSFP56-DD is qsfp56_dd, etc). One of such file is provided per form-factor, and the this provides an the concrete class implementation of ext_media_handler_base.py
		</td>
	</tr>
	<tr>
		<td valign="top">ext_media_common.py </td>
		<td>This is used for common routines and data structures shared across the files. This file also contains convenience routines such as conversions and formatters used by multiple handlers, in order to ensure the handlers do not depend on one another </td>
	</tr>
	<tr>
		<td valign="top">ext_media_api.py</td>
		<td>Provides the methods exposed to the outside world for getting the attribute dictionary, and whatever else may come in future </td>
	</tr>
	<tr>
		<td valign="top">ext_media_vendor_remap.py</td>
		<td>Provides a way update/change the parser-generated media attributes based on a custom defined map of vendor_part_number attribute, to whatever else needs changing. This is useful when a vendor wants to override the standard parsing for specific part, for whatever reason. This was provided due to the myriad non-standard media which have an aberrant field or two. This file's functionalities are executed after the parser has completed. </td>
	</tr>
</table>


##### Static Attributes
We have found empirically that regardless of the media type, all the identification information needed is contained in the first 256 bytes of the EEPROM at i2c addr 0x50 (without any paging). So in order to avoid multiple reads, we cache the first 256 bytes of each connected media, and then reference the cache for the parsing.
Ideally we would want to read only what we need, but we found it much slower to read 8 random locations of 1 byte than to read the whole 256 byte array in Python at 80khz i2c clock rate.

We have established that the routines for providing the attrs for each form factor is contained in a file called `ext_media_handler_ff_name.py`, where `ff_name` is the form-factor codified name. So the ff_name for SFP+ is `sfp_plus`, and QSFP56-DD is `qsfp56_dd` etc.
Each form-factor handler must implement a class which conforms to class `media_info` defined in `ext_media_handler_base.py`.

Class `media_info` contains all definitions for supported getter functions, and all getter functions 
All functions in `media_info` currently take two arguments: are of the form `get_attr_name`, where `attr_name` is the name of the attribute of interest.
Example:
For an attribute called `vendor_name`, the handler method in `media_info` will be named `get_vendor_name`.
The resulting key in the dictionary will then be `vendor_name`

##### Dynamic Attributes
Currently there is only one dynamic attribute `qsa_adapter` but more are coming in future.
There is currently no structured way of reading dynamic attributes since we have not formalized the behavior.
For `qsa_adapter` we simply check if the form-factor is single lane in a multi-lane port.



#### Attribute Definitions
Sample output dict for 400G optic
~~~
vendor_serial_number           : CN04HQ0098AB005               
vendor_revision                : A0                            
vendor_name                    : DELL EMC                      
display_name                   : QSFP56-DD 400GBASE-SR8-AOC-10.0M
vendor_oui                     : AC-4A-FE                      
vendor_part_number             : 9GMDY                         
vendor_date_code               : 2019-11-07                    
cable_type                     : AOC                           
special_fields                 : N/A                          
power_rating_max               : 4.5                           
cable_breakout                 : 1x1                           
lane_count                     : 8                             
connector_type                 : No separable connector        
qsa_adapter                    : N/A                           
form_factor                    : QSFP56-DD                     
media_interface                : SR                            
cable_length_detailed          : 10.0      
~~~


The attribute definitions are self-explanatory, with a few exceptions:

##### display_name
This is a very important string providing a summary of the media properties, in a way which is vendor and revision independent. Just as there are various manufacturers of 'four-door pickup trucks', there can be various manufacturers of media which share the same display_name.  However 'four-door pickup truck' is a sufficiently functionally descriptive title for what the vehicle is. Rather than pass around a bunch of attributes, we will use this string in show commands and anywhere else where specificity is not required.

The display_names for media are based on very strict rules in order to have a uniform way of referencing media. The vast majority of media will use these names, however due to some media having colloquial names and/or for historical reasons, the names can and will be overridden for certain media. Overrides can be done using the `vendor_remap` functionality or in the handler function for the given form factor. See `get_display_string` in `ext_media_handler_qsfp56_dd.py` for an example of overriding a display string.

Example of display_name: `QSFP56-DD 4x(100GBASE-SR4-ACC)-5.0M`
We will use the above example to explain the naming rules.
The rules for media naming are defined below based on pseudo-E-BNF rules:

`DISPLAY_NAME` = `FORM_FACTOR_PART`, __SPACE__, `MEDIA_TRAITS_PART`, [__HYPHEN__, `LENGTH_PART`], [__SPACE__,`EXTRA_INFO_PART`]
Output: __QSFP56-DD 4x(100GBASE-SR4-ACC)-5.0__

`FORM_FACTOR_PART` = "SFP" | "SFP+" | "SFP28" | "SFP56-DD" | "QSFP+" | "QSFP28" | "QSFP28-DD" | "QSFP56-DD" | . . .
Output: __QSFP56-DD__

`MEDIA_TRAITS_PART` = `BREAKOUT_FORM` | `STRAIGHT_FORM`
Output: __4x(100GBASE-SR4-ACC)__ this is a breakout cable, so use breakout form

`SPEED` = `FULL_SPEED`/`NUM_FAR_ENDS`
Output: __100G__ this is the speed per far end since this is a breakout

`STRAIGHT_FORM` = `SPEED`, "BASE", __HYPHEN__, `MEDIA_INTERFACE`, [`LANE_COUNT_PER_FAR_END`], [`CABLE_TYPE`]
Output: __100GBASE-SR4-ACC__

`BREAKOUT_FORM` = `NUM_FAR_ENDS`, "x(", `STRAIGHT_FORM`, ")"
Output: __4x(100GBASE-SR4-ACC)__ breakout form puts the far end in parentheses

`LENGTH_PART` = __WHOLE_PART__, "__PERIOD__", __FRACTIONAL_PART__, "M"
Output: __5.0M__ 

`MEDIA_INTERFACE` = "SR" | "CR" | "DR" | "SX" | "T" | .......
Output: __SR__

`LANE_COUNT_PER_FAR_END` = 1 | 2 | 4 | 8 | 16
Output: __4__ this is a module with 4 lane
 

__SPACE__ = " "
__HYPHEN__ = "-"
__PERIOD__ = "."
__WHOLE_PART__ = whole part of length
__FRACTIONAL_PART__ = fractional part of length, or 0. 
 
`LENGTH_PART`: This is the cable length in 1 decimal place. Example 2.0M, 3.5M, 12.0M
`LANE_COUNT_PER_FAR_END`: For straight cables or separable media , this is always 1. When 1, this value is not shown
`NUM_FAR_ENDS`: Number of cable breakouts/far-ends. Typically total lane count = NUM_FAR_ENDS * 
`SPEED`: This is the speed per breakout. For 1GigE or less, this is represented in Megabits/s (units omitted). Else in Gigabits/s
`EXTRA_INFO_PART`: Additional info such as QSA adapter, high power, etc


##### cable_length_detailed
This field allows for fractional lengths and enforces one decimal place, with a default of 0.

##### special_fields
For use when additional info such as bit error rate (coded as BER) is needed. We have not fully quantified the scope of this field yet.




### Proposed format
The intention is to maintain existing field names and their contents, for backward-compatibility.  New fields shall be added.

The following shall be added to the dictionary:

<h4>cable_breakout</h4>
<table style="border-style:none;">
	<tr>
		<td valign="top">Description</td>
		<td>Breakout of attached cable, if any, i.e. the number of cables directly attached to transceiver</td>
	</tr>
	<tr>
		<td valign="top">Value(s)</td>
		<td>integer; 0 when no directly-attached cable</td>
	</tr>
	<tr>
		<td valign="top">Needed by</td>
		<td>Validation of port breakout configurations, i.e. whether or not transceiver supports breakout</td>
	</tr>
</table>
<h4>cable_length_detailed</h4>
<table style="border-style:none;">
	<tr>
		<td valign="top">Description</td>
		<td>Length of directly-attached cable, in metres</td>
	</tr>
	<tr>
		<td valign="top">Value(s)</td>
		<td>float; 0.0 when no directly-attached cable</td>
	</tr>
	<tr>
		<td valign="top">Needed by</td>
		<td>Layer-1 configuration lookup, e.g preemphasis</td>
	</tr>
	<tr>
		<td valign="top">Notes</td>
		<td>Complements existing cable_length field, can represent lengths < 1 m or fractional lengths</td>
	</tr>
</table>
<h4>cable_type</h4>
<table style="border-style:none;">
	<tr>
		<td valign="top">Description</td>
		<td>Type of directly-attached cable</td>
	</tr>
	<tr>
		<td valign="top">Value(s)</td>
		<td>string, one of "DAC", "RJ45", "FIBER", "AOC", "ACC"</td>
	</tr>
	<tr>
		<td valign="top">Needed by</td>
		<td>Layer-1 configuration lookup, e.g NPU interface type</td>
	</tr>
</table>
<h4>display_name</h4>
<table style="border-style:none;">
	<tr>
		<td valign="top">Description</td>
		<td>String suitable for displaying description of transceiver</td>
	</tr>
	<tr>
		<td valign="top">Value(s)</td>
		<td>string</td>
	</tr>
	<tr>
		<td valign="top">Needed by</td>
		<td>CLI show command(s)</td>
	</tr>
	<tr>
		<td valign="top">Notes</td>
		<td>Rather than the CLI code attempt to construct descriptive string for an installed transceiver, from the information in the TRANSCEIVER_TABLE, xcvrd can construct a short descriptive string for use by CLI show commands.  Since transceiver "knowledge" is inside xcvrd, it is reasonable for it to compose the displayable descriptive string, rather than the CLI.</td>
	</tr>
	<tr>
		<td valign="top">Examples</td>
		<td>"SFP+ 10GBASE-DWDM-TUNABLE", "QSFP28 100GBASE-CR4-0.5M"</td>
	</tr>
</table>
<h4>lane_count</h4>
<table style="border-style:none;">
	<tr>
		<td valign="top">Description</td>
		<td>Number of lanes</td>
	</tr>
	<tr>
		<td valign="top">Value(s)</td>
		<td>integer, 1..N</td>
	</tr>
	<tr>
		<td valign="top">Needed by</td>
		<td>Validation of port breakout configurations, i.e. whether or not transceiver supports breakout</td>
	</tr>
</table>
<h4>media_interface</h4>
<table style="border-style:none;">
	<tr>
		<td valign="top">Description</td>
		<td>Media interface</td>
	</tr>
	<tr>
		<td valign="top">Value</td>
		<td>string, one of "SR", "CR", "ZR", "DR", "EDR", "BASE-T"</td>
	</tr>
	<tr>
		<td valign="top">Needed by</td>
		<td>Layer-1 configuration lookup, e.g preemphasis</td>
	</tr>
</table>
<h4>form_factor</h4>
<table style="border-style:none;">
	<tr>
		<td valign="top">Description</td>
		<td>Detailed transceiver type, i.e. form factor</td>
	</tr>
	<tr>
		<td valign="top">Value(s)</td>
		<td>string, one of "SFP", "SFP+", "SFP28", "QSFP+", "QSFP28", "QSFP28-DD", "QSFP56-DD", "RJ45"</td>
	</tr>
	<tr>
		<td valign="top">Needed by</td>
		<td>Layer-1 configuration lookup, e.g preemphasis</td>
	</tr>
	<tr>
		<td valign="top">Notes</td>
		<td>This field is intended as a replacement for the existing "type_abbrv_name" field.  That field does not fully discriminate a transceiver "type".  For example, a QSFP-DD transceiver may be a QSFP28-DD or QSFP56-DD.  Also, other types, such as "SFP28", are missing.<br/>Rather than introduce new values, such as "SFP28", "QSFP-DD28" and "QSFP-DD56" for the existing field, and deprecating old values, such as "QSFP-DD", a new field will be introduced, that fully discriminates a "type", and the old field shall be deprecated.</td>
	</tr>
</table>
