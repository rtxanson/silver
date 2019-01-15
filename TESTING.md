# Test processes

The intent of this document is to list the minimal steps to initiate
server-side processes to test integrations from the client side.

## Creating an Auth Token

For now the application uses basic token authentication. In order to create or
retrieve a token, log in to the Django admin interface, click on Tokens, and
add or copy an existing token.

## Authenticating

Authenticated views require an `Authorization` header of the following
structure:

    Authorization: Token YOUR_TOKEN_HERE


## Triggering a payment to process

This is a precise process that relies on API requests, and automatic
server-side processes.

### Create a Customer

Send the following request, noting the customer ID returned:

    curl --request POST \
      --url http://dev.billing.dynamicic.com/silver/customers/ \
      --header 'authorization: Token $YOUR_AUTH_TOKEN' \
      --header 'content-type: application/json' \
      --data '{
        "customer_reference": "bbqasddf",
        "first_name": "Bbq",
        "last_name": "Asdf",
        "company": "Some Jumbo Company",
        "email": "asdf@bbq.com",
        "address_1": "1234 Mulberry Lane",
        "address_2": "",
        "city": "Nantucket",
        "state": "Hawaii",
        "zip_code": "41414",
        "country": "US",
        "currency": "USD",
        "phone": "",
        "extra": "",
        "sales_tax_number": "",
        "sales_tax_name": "",
        "sales_tax_percent": null,
        "consolidated_billing": false,
        "meta": {
            "cardNumber": "4111111111111111",
            "cardCode": "123",
            "expirationDate": "2020-12"
        }
    }'

### Create a Customer Payment Method

Send the following request, substituting `CUSTOMER_ID` with the new customer id.

    curl --request POST \
      --url http://dev.billing.dynamicic.com/silver/customers/CUSTOMER_ID/payment_methods/ \
      --header 'authorization: Token $YOUR_AUTH_TOKEN' \
      --header 'content-type: application/json' \
      --data '{
        "payment_processor_name": "authorizenet_triggered",
        "verified": true,
        "canceled": false,
        "valid_until": "2019-10-12T21:33:56.145656Z",
        "display_info": "testing"
    }'

### Create an invoice provider

TODO: document

### Create a (draft) Invoice

This is a little wonky for now, because it relies on Django REST API's
`CustomerUrl` serializer, so you will need to create URLs to related objects
instead of simply referencing the object's id.

Create the invoice with the following request and note the invoice id.

    curl --request POST \
      --url http://dev.billing.dynamicic.com/silver/invoices/ \
      --header 'authorization: Token $YOUR_AUTH_TOKEN' \
      --header 'content-type: application/json' \
      --data '{
            "series": "InvoiceSeriesB",
            "provider": "http://$HOSTNAME/silver/providers/$PROVIDER_ID/",
            "customer": "http://$HOSTNAME/silver/customers/$CUSTOMER_ID/",
            "due_date": "2019-03-01",
            "issue_date": "2019-02-15",
            "paid_date": null,
            "cancel_date": null,
            "sales_tax_name": "sales tax",
            "sales_tax_percent": "0.05",
            "currency": "USD",
            "transaction_currency": "USD",
            "transaction_xe_rate": "1.0000",
            "transaction_xe_date": "2019-01-15",
            "state": "draft",
            "proforma": null,
            "invoice_entries": [
                {
                    "description": "Charcoal Latte",
                    "unit": "Cup",
                    "unit_price": "25.0000",
                    "quantity": "2.0000",
                    "total_before_tax": 50.0,
                    "start_date": null,
                    "end_date": null,
                    "prorated": false,
                    "product_code": null
                }
            ]
        }'

### Set the invoice to Issued

Issuing the invoice will automatically create a Payment object associated with
the Invoice and Customer.

    curl --request PUT \
      --url http://dev.billing.dynamicic.com/silver/invoices/$INVOICE_ID/state/ \
      --header 'authorization: Token $YOUR_AUTH_TOKEN' \
      --header 'content-type: application/json' \
      --data '{
          "state": "issued"
      }'

### Run the payment process

This should be running on a cron or [ celery task ][celery], but for now:

1. Log into the server
2. Activate the virtual environment
3. run `./manage.py execute_transactions`

  [celery]: https://github.com/silverapp/silver/blob/0009ff4ca52dfc711e2f160ad90b449060fc4007/settings.py
