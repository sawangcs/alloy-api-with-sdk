# Connecting to Alloy API using OCI SDK

This document provides a step-by-step guide to connect to the Alloy API using the Oracle Cloud Infrastructure (OCI) SDK.

## Prerequisites

1. **Oracle Cloud Infrastructure (OCI) Account**: Ensure you have an active OCI account.
2. **[OCI SKD](https://docs.oracle.com/en-us/iaas/Content/API/Concepts/apisigningkey.htm#Required_Keys_and_OCIDs)**: Install the OCI SDK for your preferred programming language (e.g., .Net, Java, Python, etc.).
3. **[API Key](https://docs.oracle.com/en-us/iaas/Content/API/Concepts/apisigningkey.htm)**: Generate an API key for authentication.

## Step 1: Install OCI SDK

Install the OCI SDK for your preferred programming language. To install the OCI SDK, use the following urls:

- [SDK Guide for JAVA](https://docs.oracle.com/en-us/iaas/Content/API/SDKDocs/javasdkgettingstarted.htm)
- [SDK Guide for Python](https://docs.oracle.com/en-us/iaas/tools/python/2.144.0/configuration.html)
- [SDK Guide for Ruby](https://docs.oracle.com/iaas/tools/ruby/latest/index.html#label-Configuring+the+SDK)
- [SDK Guide for Go](https://github.com/oracle/oci-go-sdk/blob/master/README.md#configuring)
- [SDK Guide for TypeScript and JavaScript](https://docs.oracle.com/en-us/iaas/Content/API/SDKDocs/typescriptsdkgettingstarted.htm#configuring)
- [SDK Guide for .Net](https://docs.oracle.com/en-us/iaas/Content/API/SDKDocs/dotnetsdkgettingstarted.htm#Configure)

## Step 2: Configure OCI SDK

Create a configuration file to store your OCI credentials. The default location for the configuration file is `~/.oci/config`.

Example configuration file:

```ini
[DEFAULT]
user=ocid1.user.oc1..exampleuniqueID
fingerprint=20:3b:97:13:55:1c:2b:aa:3e:4b:1d:51:1e:1a:1f:1a
key_file=~/.oci/oci_api_key.pem
tenancy=ocid1.tenancy.oc1..exampleuniqueID
region=us-ashburn-1
```

## Step 3: Generate API Key

Generate an API key and save the private key to a file (e.g., `oci_api_key.pem`). Ensure the key file has the correct permissions use the following url:

- [Registering Key and OCIDs](https://docs.oracle.com/en-us/iaas/Content/API/Concepts/apisigningkey.htm)

## Step 4: Connect to Alloy API

Use the OCI SDK to connect to the Alloy API. Below is an example in C# .Net Standard:

```c#
const string OciConfigProfileName = "DEFAULT";
private static NLog.Logger logger = NLog.LogManager.GetCurrentClassLogger();
const string userName = "oci-dotnetsdk-test-user";

public static async Task Main()
{
    string compartmentId = "{compartmentId}";
    string priceListId = "{priceListId}";

    // Accepts profile name and creates a auth provider based on config file
    var provider = new ConfigFileAuthenticationDetailsProvider("../.oci/config", OciConfigProfileName);
    //new ConfigFileAuthenticationDetailsProvider(OciConfigProfileName);

    // Create a client for the service to enable using its APIs
    var client = new PricingClient(provider, new ClientConfiguration(), "https://subscription-pricing.us-dcc-swjordan-2.oci.oraclecloud28.com/");

    try
    {
        await ListPriceLists(client, compartmentId);
        await ListPriceListItems(client, compartmentId, priceListId);
    }
    catch (Exception e)
    {
        logger.Info($"Received exception due to {e.Message}");
    }
    finally
    {
        client.Dispose();
    }
}
```

## Step 5: Make API Requests

Use the Alloy client to make API requests. Refer to the Alloy API documentation for available endpoints and request parameters.

```c#
private readonly RetryConfiguration retryConfiguration;
private const string basePathWithoutHost = "/20230101";

/// <summary>
/// Creates a new service instance using the given authentication provider and/or client configuration and/or endpoint.
/// A client configuration can also be provided optionally to adjust REST client behaviors.
/// </summary>
/// <param name="authenticationDetailsProvider">The authentication details provider. Required.</param>
/// <param name="clientConfiguration">The client configuration that contains settings to adjust REST client behaviors. Optional.</param>
/// <param name="endpoint">The endpoint of the service. If not provided and the client is a regional client, the endpoint will be constructed based on region information. Optional.</param>
public PricingClient(
    IBasicAuthenticationDetailsProvider authenticationDetailsProvider, 
    ClientConfiguration clientConfiguration = null, string endpoint = null)
    : base(authenticationDetailsProvider, clientConfiguration)
{
    if (!DeveloperToolConfiguration.IsServiceEnabled("SUBSCRIPTION-PRICING"))
    {
        throw new ArgumentException("The DeveloperToolConfiguration disabled this service, this behavior is controlled by DeveloperToolConfiguration.OciEnabledServiceSet variable. Please check if your local DeveloperToolConfiguration file has configured the service you're targeting or contact the cloud provider on the availability of this service");
    }
    service = new Service
    {
        ServiceName = "SUBSCRIPTION-PRICING",
        ServiceEndpointPrefix = "subscription-pricing",
        ServiceEndpointTemplate = "https://subscription-pricing.{region}.oci.{secondLevelDomain}"
    };

    ClientConfiguration clientConfigurationToUse = clientConfiguration ?? new ClientConfiguration();

    if (authenticationDetailsProvider is IRegionProvider)
    {
        // Use region from Authentication details provider.
        SetRegion(((IRegionProvider)authenticationDetailsProvider).Region);
    }

    if (endpoint != null)
    {
        logger.Info($"Using endpoint specified \"{endpoint}\".");
        SetEndpoint(endpoint);
    }

    this.retryConfiguration = clientConfigurationToUse.RetryConfiguration;
}

/// <summary>
/// Get the PriceList record with significant fields for the priceList Id passed..
/// </summary>
/// <param name="request">The request object containing the details to send. Required.</param>
/// <param name="retryConfiguration">The retry configuration that will be used by to send this request. Optional.</param>
/// <param name="cancellationToken">The cancellation token to cancel this operation. Optional.</param>
/// <param name="completionOption">The completion option for this operation. Optional.</param>
/// <returns>A response object containing details about the completed operation</returns>
public async Task<string> ListPriceLists(
    ListPriceListRequest request,
    RetryConfiguration retryConfiguration = null,
    CancellationToken cancellationToken = default,
    HttpCompletionOption completionOption = HttpCompletionOption.ResponseContentRead)
{
    logger.Trace("Called ListPriceLists");
    Uri uri = new Uri(this.restClient.GetEndpoint(), System.IO.Path.Combine(basePathWithoutHost, "/priceLists".Trim('/')));
    HttpMethod method = new HttpMethod("GET");
    HttpRequestMessage requestMessage = Converter.ToHttpRequestMessage(uri, method, request);
    requestMessage.Headers.Add("Accept", "application/json");
    GenericRetrier retryingClient = Retrier.GetPreferredRetrier(retryConfiguration, this.retryConfiguration);
    HttpResponseMessage responseMessage;

    try
    {
        Stopwatch stopWatch = new Stopwatch();
        stopWatch.Start();
        if (retryingClient != null)
        {
            responseMessage = await retryingClient.MakeRetryingCall(this.restClient.HttpSend, requestMessage, completionOption, cancellationToken).ConfigureAwait(false);
        }
        else
        {
            responseMessage = await this.restClient.HttpSend(requestMessage, completionOption: completionOption).ConfigureAwait(false);
        }
        stopWatch.Stop();
        ApiDetails apiDetails = new ApiDetails
        {
            ServiceName = "Identity",
            OperationName = "ListRegions",
            RequestEndpoint = $"{method.Method} {requestMessage.RequestUri}",
            ApiReferenceLink = "",
            UserAgent = this.GetUserAgent()
        };
        this.restClient.CheckHttpResponseMessage(requestMessage, responseMessage, apiDetails);
        logger.Debug($"Total Latency for this API call is: {stopWatch.ElapsedMilliseconds} ms");
        return await Converter.FromHttpResponseMessageToJson(responseMessage);
    }
    catch (OciException e)
    {
        logger.Error(e);
        throw;
    }
    catch (Exception e)
    {
        logger.Error($"ListRegions failed with error: {e.Message}");
        throw;
    }
}
```

- **API Region and Second Level Domain**
  - Region : us-dcc-swjordan-2
  - Second Level Domain: oraclecloud28.com

- **API Prefixs**
  
  **Price List API Prefix**
  - https://subscription-pricing.{region}.oci.{secondLevelDomain}
  
  **Billing API Prefix**
  - https://billing.{region}.oci.{secondLevelDomain}

## Conclusion

By following these steps, you can successfully connect to the Alloy API using the OCI SDK. Ensure you refer to the official OCI SDK and Alloy API documentation for more details and advanced usage.
