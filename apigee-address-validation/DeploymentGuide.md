

## Apigee Setup:

1. Provision Apigee using Terraform sample https://github.com/apigee/terraform-modules/tree/main/samples/x-l7xlb  
     
1. Install apigecli

```
curl -L https://raw.githubusercontent.com/apigee/apigeecli/main/downloadLatest.sh | sh -
export PATH=$PATH:$HOME/.apigeecli/bin
```

1. Create Google Map API Key

```
sudo apt-get install jq
json_output=$(gcloud beta services api-keys create --display-name="My MAP API Key" --format=json)

GMAPS_KEY=$(echo $json_output | jq -r '.response.keyString')
```

1. Create Service Account

```
gcloud iam service-accounts create addressvalidationsa \
    --description="Address Validation SA" \
    --display-name="Address Validation SA"

```

1. Deploy Apigee Assets

```
export PROJECT="your-project-id"
export REGION=REGION
export APIGEE_ENVIRONMENT=eval

gcloud config set project $PROJECT
gcloud services enable geocoding-backend.googleapis.com 
gcloud services enable addressvalidation.googleapis.com 

```

1. Store Map API Keys to Apigee KVM

```
apigeecli kvms create -e $APIGEE_ENVIRONMENT -n address-gmap-keys -o $PROJECT -t $(gcloud auth print-access-token)

apigeecli kvms entries create -m address-gmap-keys -k gmaps_key -l $GMAPS_KEY -e $APIGEE_ENVIRONMENT -o $PROJECT -t $(gcloud auth print-access-token)
```

1. Deploy Apigee Proxy

```
git clone https://github.com/echarish/ApigeeSamples.git
cd ApigeeSamples/apigee-address-validation

apigeecli apis import -o $PROJECT -f . -t $(gcloud auth print-access-token)

apigeecli apis deploy -n AddressValidation-Service-v1 -o $PROJECT -e $APIGEE_ENVIRONMENT -t $(gcloud auth print-access-token) -s addressvalidationsa@$PROJECT.iam.gserviceaccount.com --ovr

```

1. Create API Product

```
export PRODUCT_NAME="AddressValidation-API-v1"
apigeecli products create -n "$PRODUCT_NAME" \
  -m "$PRODUCT_NAME" \
  -o "$PROJECT" -e "$APIGEE_ENVIRONMENT" \
  -f auto -p "AddressValidation-Service" -t $(gcloud auth print-access-token) 

```

1. Create a Developer

```
export DEVELOPER_EMAIL="example-developer@cymbalgroup.com"

apigeecli developers create -n "$DEVELOPER_EMAIL" \
  -f "Example" -s "Developer" \
  -u "$DEVELOPER_EMAIL" -o "$PROJECT" -t $(gcloud auth print-access-token)

#Now let’s create an app subscription using the test developer account.

export APP_NAME=addressvalidation-app-1

DEVELOPER_APP=$(apigeecli apps create --name "$APP_NAME" \
  --email "$DEVELOPER_EMAIL" \
  --prods "$PRODUCT_NAME" \
  --org "$PROJECT" --token $(gcloud auth print-access-token))

API_KEY=$(echo $DEVELOPER_APP | jq -r '.credentials[0].consumerKey')
```

1. Use API\_KEY to make call to Address Valdiation proxy deployed on Apigee

```

export APIGEE_URL=SET_YOUR_APIGEE_URL

curl --location '$APIGEE_URL/v1/addressvalidation?apikey=$API_KEY' \
--header 'Content-Type: application/json' \
--data '{
    "address": {
        "regionCode": "US",
        "locality": "Mountain View",
        "addressLines": [
            "1600 Amphitheatre Pkwy"
        ]
    }
}'
```
