# Label LIVE

Integrate a label printer via [Label LIVE](https://label.live/)'s [HTTP REST API](https://label.live/guides/automated-label-printing-integration-with-production-processes).

## Example Printer Configuration

{% hint style="info" %}
**You'll need to do the following for this configuration to work:**

1. Replace the `choices` in `options.host` to the address of your computer that is running Label LIVE.
2. Download the [Sample Label Designs](label-live.md#sample-label-designs), open them with Label LIVE and pin them to the home screen.
3. Replace `your-printer-id` in the config to your actual printer name in Label LIVE. ([How to find your printer ID](label-live.md#how-to-find-your-printer-id)?)
{% endhint %}

```javascript
{
  options: {
    // You should keep Label LIVE running on the machine to respond to API requests
    host: {
      type: 'string',
      choices: [
        'http://192.168.88.123:11180',
        'https://example.com',
        // ...
      ],
      saveLastValue: true,
    },
    // Get these sample designs in the Sample Label Designs section below
    design: {
      enum: [
        '70×20 Two Lines with QR Code',
        '70×20 QR Code with Asset Reference Number',
        // ...
      ],
      saveLastValue: true,
    },
    use_preview_printer: {
      type: 'boolean',
      default: false,
      saveLastValue: false,
    },
  },
  getLabel: ({ item, options, utils }) => {
    switch (options.design) {
      case '70×20 Two Lines with QR Code': {
        const itemName = item.name.trim();
        const { nextLine: line_1, remaining: line_2 } = utils.getNextLine(
          itemName,
          18,
        );
        return {
          LINE_1: line_1.trim() || ' ',
          LINE_2: line_2.trim() || ' ',
          SIDE: item.collection.name,
          QR_CODE_DATA: item.individual_asset_reference
            ? `z-inv://iar/${item.individual_asset_reference}`
            : `z-inv://item/${item.__id}`,
        };
      }
      case '70×20 QR Code with Asset Reference Number': {
        return {
          ASSET_REFERENCE_NUMBER: item.individual_asset_reference || ' ',
          QR_CODE_DATA: item.individual_asset_reference
            ? `z-inv://iar/${item.individual_asset_reference}`
            : `z-inv://item/${item.__id}`,
        };
      }
      default: {
        throw new Error(`Design not handled in getLabel: "${options.design}"`);
      }
    }
  },
  getPreview: ({ options, label, utils }) => {
    const size = (() => {
      switch (options.design) {
        case '70×20 Two Lines with QR Code': {
          // You can open a preview image in the browser to get its size.
          return {
            width: 827,
            height: 237,
          };
        }
        case '70×20 QR Code with Asset Reference Number': {
          // You can open a preview image in the browser to get its size.
          return {
            width: 827,
            height: 237,
          };
        }
        default: {
          throw new Error(
            `Design not handled in getPreview: "${options.design}"`,
          );
        }
      }
    })();

    return {
      uri: [
        options.host,
        '/api/v1/print?design=',
        encodeURIComponent(options.design),
        '&variables=',
        encodeURIComponent(JSON.stringify(label)),
      ].join(''),
      ...size,
    };
  },
  print: ({ options, labels, utils, signal }) => {
    const { host, design } = options;

    // With multiple printers connected to the same computer,
    // you can support printing designs on multiple sizes of labels.
    const designPrinterMap = {
      '70×20 Two Lines with QR Code': 'your-printer-id',
      '70×20 QR Code with Asset Reference Number': 'your-printer-id',
    };

    const printer = options.use_preview_printer
      ? 'Preview'
      : designPrinterMap[options.design];

    return Promise.all(
      labels.map(label => {
        return fetch(
          [
            options.host, 
            '/api/v1/print?design=',
            encodeURIComponent(options.design),
            '&variables=',
            encodeURIComponent(JSON.stringify(label)),
            '&printer=',
            printer,
            '&copies=1',
          ].join(''),
          {
            signal /* this is used to let the user abort the fetch */,
            method: 'POST',
          },
        )
          .then(response => response.json())
          .then(json => {
            if (!json.success) {
              // Error will be shown on the UI
              throw new Error(json.error);
            }
          });
      }),
    );
  },
}
```

## Sample Label Designs

{% hint style="info" %}
**You'll need to pin the designs on the Home Screen of Label LIVE before using them via API**\
![](<../../.gitbook/assets/Screen Shot 2023-09-29 08.04.33 AM (Label LIVE)@2x (2).png>)
{% endhint %}

{% file src="../../.gitbook/assets/70×20 Two Lines with QR Code.lsc" %}

{% file src="../../.gitbook/assets/70×20 QR Code with Asset Reference Number.lsc" %}

## Troubleshooting & FAQ

<details>

<summary>How to find your printer ID</summary>

1. Open a design in Label LIVE and select the "Print" tab.
2. Hover the printer in the drop-down menu.
3. Click "Copy Printer ID".

![](<../../.gitbook/assets/Screen Shot 2023-09-29 10.44.52 AM (Label LIVE)@2x (1).png>)



</details>

<details>

<summary>The preview is showing, but when I press "Print" it does not respond</summary>

There might be an error alert box blocking Label LIVE's API response. Please check your computer.

![](<../../.gitbook/assets/Screen Shot 2023-09-29 11.08.55 AM (Arc)@2x.png>)

</details>

