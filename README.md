# DPD-ShipmentService

A C# client for the Shipment Service of the DPD Web Services - create a parcel label for a wide range of DPD products and services quickly and simply.

- *Current specifications, endpoints, URLs can be found within the [DPD development portal](https://esolutions.dpd.com/entwickler/dpdwebservices.aspx)*. The website provides up-to-date documentation, code examples as well as a sandbox for development purposes. Please register as a developer to get access to all detailed information.
- *Each development using the DPD Web Connect service must be validated and approved by DPD*. This validation is mandatory to get started with the DPD Web Connect service. The correct handling, the implemented services as well as products must be tested and validated prior any access to the production environment.

| ⚠️ These are Kaupisch IT-Systeme GmbH's own code examples and not official DPD Germany code examples. |
| - |

## Sample


## Create proxy classes

The services' proxy code was generated with `wsdl.exe` and enhanced with [fancywsdl.exe](https://github.com/Kaupisch-IT/FancyWsdl):

- Login Service v2.0

  ```console
  wsdl.exe https://public-ws-stage.dpd.com/services/LoginService/V2_0/?wsdl
  fancywsdl.exe https://public-ws-stage.dpd.com/services/LoginService/V2_0/?wsdl https://public-ws-stage.dpd.com/services/LoginService/V2_0/?xsd=Authentication_2_0.xsd https://public-ws-stage.dpd.com/services/LoginService/V2_0/?xsd=LoginService-Public_2_0.xsd
  ```
 
- Shipment Service v4.3

  ```console
  wsdl.exe https://public-ws-stage.dpd.com/services/ShipmentService/V4_3/?wsdl
  fancywsdl.exe DpdShipmentService_4_3.cs https://public-ws-stage.dpd.com/services/ShipmentService/V4_3/?wsdl https://public-ws-stage.dpd.com/services/ShipmentService/V4_3/?xsd=ShipmentService-Public_4_3.xsd https://public-ws-stage.dpd.com/services/ShipmentService/V4_3/?xsd=Authentication_2_0.xsd
  ```

- Shipment Service v4.4

  ```console
  wsdl.exe https://public-ws-stage.dpd.com/services/ShipmentService/V4_4/?wsdl
  fancywsdl.exe DpdShipmentService_4_4.cs https://public-ws-stage.dpd.com/services/ShipmentService/V4_4/?wsdl https://public-ws-stage.dpd.com/services/ShipmentService/V4_4/?xsd=ShipmentService-Public_4_4.xsd https://public-ws-stage.dpd.com/services/ShipmentService/V4_4/?xsd=Authentication_2_0.xsd
  ``` 
