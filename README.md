# DPD-ShipmentService

A C# client for the Shipment Service of the DPD Web Services - create a parcel label for a wide range of DPD products and services quickly and simply.

- *Current specifications, endpoints, URLs can be found within the [DPD development portal](https://esolutions.dpd.com/entwickler/dpdwebservices.aspx)*. The website provides up-to-date documentation, code examples as well as a sandbox for development purposes. Please register as a developer to get access to all detailed information.
- *Each development using the DPD Web Connect service must be validated and approved by DPD*. This validation is mandatory to get started with the DPD Web Connect service. The correct handling, the implemented services as well as products must be tested and validated prior any access to the production environment.

| :warning: These are Kaupisch IT-Systeme GmbH's own code examples and not official DPD Germany code examples. |
| - |

## Sample

Create service shipment data - Versandaufträge zusammenstellen:

```csharp
Address sender = new Address
{
   Name1 = "DingeVersand GmbH",
   Street = "Musterstraße",
   HouseNo = "123",
   ZipCode = "01234",
   City = "Musterstadt a.d. Muster",
   Country = "DE",
};
Address collectionRequestAddress = sender;

List<ShipmentServiceData> orders = new List<ShipmentServiceData>();
// foreach (...) // über alle zu erstellenden Versandaufträge iterieren
{
   string identificationNumber = "ABC/DEF/123";
   string customerReference1 = "myKundenreferenz1";
   string customerReference2 = "myKundenreferenz2";

   Address recipient = new Address
   {
      Name1 = "Maxi Mustermensch",
      Name2 = "c/o Müllermeierschmidt",
      Street = "Musterweg",
      HouseNo = "456",
      ZipCode = "45678",
      City = "Musterhausen",
      Country = "DE",
      Email = "maxi.mustermensch@example.com",
   };

   int parcelCount = 2; // 2 Packstücke
   long weight = 300; // ~3kg Gesamtgewicht

   GeneralShipmentDataProduct product = GeneralShipmentDataProduct.CL; // ~DPD Classic

   orders.Add(new ShipmentServiceData
   {
      // allgemeine Versanddaten
      GeneralShipmentData = new GeneralShipmentData()
      {
         IdentificationNumber = identificationNumber, // brauchen wir, um den Paketschein später wieder einem Versandauftrag zuordnen zu können

         // Produkt, Absender, Empfänger
         Product = product,
         Sender = sender,
         Recipient = recipient,

         // ("MPS" = "Multi Parcel Shipping", Sendung mit mehreren Paketen)
         MpsWeight = weight,
         MpsCompleteDelivery = (parcelCount>1),
         MpsCompleteDeliveryLabel = true,

         // Kundenreferenz-Daten (hier: BelegID und Kundennummer)
         MpsCustomerReferenceNumber1 = customerReference1,
         MpsCustomerReferenceNumber2 = customerReference2,
      },
      
      // Pakete der Sendung (so viele Parcel-Objekte erstellen, wie in parcelCount angegeben)
      Parcels = Enumerable.Range(0,parcelCount).Select(idx => new Parcel
      {
         Weight = (idx==0) ? weight : (int?)null, // MpsWeight wurde angegeben, auf den Paketscheinen würde dann aber überall "0 kg" draufstehen
      })
      .ToArray(),

      // Angaben zum Produkt/gewünschten DPD-Service
      ProductAndServiceData = new ProductAndServiceData()
      {
         // Paket abholen lassen
         OrderType = ProductAndServiceDataOrderType.Consignment,

         // Daten für die Abholung
         Pickup = new Pickup()
         {
            CollectionRequestAddress = collectionRequestAddress, // Adresse, an der die Sendungen abgeholt werden sollen
            Quantity = parcelCount,
            FromTime1 = 0800, // (notwendig)
            ToTime1 = 1700,   // (notwendig)
         },

         // Benachrichtigung über Versandstatus
         Predict = (String.IsNullOrWhiteSpace(recipient.Email))
            ? null
            : new Notification()
            {
               Channel = 1, // = "Email"
               Value = recipient.Email,
               Language = "DE",
            },
      }
   });
}
```

Store orders - für die Auswahl an Versandaufträgen eine Bestellung auslösen

```csharp
//
// DPD-Bestellung durchführen
//
PrintOptionPaperFormat printOptionPaperFormat = PrintOptionPaperFormat.A4;
StartPosition labelStartPosition = StartPosition.UPPER_LEFT;

// Login-Daten für den DPD-Webservice holen
string delisID = Settings.Default.DelisID;
string password = Settings.Default.DelisPassword;
string messageLanguage = "de_DE";

// beim LoginService wird angemeldet und ein Token geholt; beim ShipmentService können die Paketscheine bestellt werden
LoginServicePublic_2_0 loginService = new LoginServicePublic_2_0 { Url = Settings.Default.DpdLoginServiceUrl };
ShipmentServicePublic_4_3 shipmentService = new ShipmentServicePublic_4_3 { Url = Settings.Default.DpdShipmentServiceUrl };

// erst einmal anmelden
Login login = loginService.GetAuth(delisID,password,messageLanguage);

// Parameter für die StoreOrders-Methode: Druck/Ausgabe-Optionen und eine Liste mit Versandaufträgen
StoreOrders dpdStoreOrders = new StoreOrders
{
   PrintOptions = new PrintOptions()
   {
      PrintOption = new[]
      {
         new PrintOption()
         {
            // A4-Seitenformat: da kommen dann bis zu vier Paketscheine drauf; A6-Seitenformat: ein Paketschein pro Seite
            PaperFormat = printOptionPaperFormat,
            StartPosition = labelStartPosition,
         }
      }
   },
   // die Versandaufträge anhängen
   Order = orders.ToArray(),
};

// die Depot-Nummer des Versenders für jeden Versandauftrag (wird benötigt)
foreach (ShipmentServiceData order in dpdStoreOrders.Order)
   order.GeneralShipmentData.SendingDepot = login.Depot;

// das Token aus dem Login von gerade eben an den ShipmentService übergeben und die StoreOrders-Methode aufrufen
// StoreOrders: Löst eine Bestellung für die Versandaufträge aus und liefert eine (einzige) PDF-Datei zurück, die alle zugehörigen Paketscheine enthält
shipmentService.AuthenticationValue = new Authentication
{
   DelisId = delisID,
   AuthToken = login.AuthToken,
   MessageLanguage = messageLanguage
};
StoreOrdersResponse response = shipmentService.StoreOrders(dpdStoreOrders);
```

Evaluate return values - speichert die PDF-Datei mit den Paketscheinen

```csharp
//
// Rückgabe auswerten
//

// Dateiname und Pfad für die PDF-Datei erzeugen
string pdfFileName = $"paketschein-{DateTime.Now:yyMMdd}-{Guid.NewGuid()}.pdf";
string pdfPath = Path.Combine(Settings.Default.PdfFolder,pdfFileName);

// falls die StoreOrders eine PDF-Datei als Content zurückgeliefert hat...
if (response.OrderResult.Output?.Content!=null)
{
   // ...die PDF-Datei (wird als byte[] zurückgegeben) in den Ordner für die PDF-Dateien abspeichern
   Directory.CreateDirectory(Path.GetDirectoryName(pdfPath));
   File.WriteAllBytes(pdfPath,response.OrderResult.Output.Content);
   Process.Start(pdfPath);
}

// die Paketschein- und Tracking-Daten für die zurückgelieferten Paketscheine ermitteln
foreach (ShipmentResponse shipmentResponse in response.OrderResult.ShipmentResponses)
{
   string identificationNumber = shipmentResponse.IdentificationNumber;
   string trackingID = shipmentResponse.MpsId;
   bool success = (shipmentResponse.Faults==null);

   Debug.WriteLine($"Result: Shipment {identificationNumber} - Tracking ID {trackingID} - Success: {success}");
}

// nach Fehlern suchen
foreach (ShipmentResponse shipmentResponse in response.OrderResult.ShipmentResponses)
{
   // nach Fehlern suchen
   if (shipmentResponse.Faults!=null)
      foreach (FaultCodeType fault in shipmentResponse.Faults)
      {
         string identificationNumber = shipmentResponse.IdentificationNumber;
         string message = fault.Message;

         Debug.WriteLine($"Error: Shipment {identificationNumber} - {message}");
      }
}
```



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
