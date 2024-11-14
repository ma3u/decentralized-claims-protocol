# Verifiable Presentation Protocol

This section defines a protocol for storing and presenting Verifiable Credentials and other identity-related
resources. The Verifiable Presentation Protocol covers the following aspects:

- Endpoints and message types for storing identity resources belonging to a [=Holder=] in a [=Credential Service=]
- Endpoints and message types for resolving an identity [=Resource=]
- Secure token exchange for restricting access to  [=Credential Service=] endpoints

<aside class="note">
The Verifiable Presentation Protocol is designed to address the problem of resolving [=Verifiable Presentations=] and
other claims when they cannot be passed as part of a client request. For example, they often cannot be included as part
of an HTTP message header due to size restrictions. The protocol provides a secure mechanism for a [=Verifier=] to 
resolve credential-related resources. The protocol also provides a mechanism for storing issued [=Verifiable Credentials=].   
</aside>

## Presentation Flow

The following sequence diagram depicts a non-normative flow where a client interacts with a [=Verifier=] to present a
[=Verifiable Credential=]:

![alt text 2](specifications/auth.flow.png "Presentation Flow")

1. The client sends a request to its [=Secure Token Service=] for a [=Self-Issued ID Token=]. The API used to make this
   request is implementation specific. The client may include a set of scopes that define the Verifiable Credentials the
   client wants the Verifier to have access to. This set of scopes is determined out of band and may be derived from
   metadata the verifier has previously made available to the client.
2. The [=Secure Token Service=] responds with the Self-Signed ID token containing a `token` claim with the value set to
   an access token. The access token can be used by the verifier to request Verifiable Credentials from the client's
   Credential Service.
3. The client makes a request to the [=Verifier=] for a protected resource and includes the [=Self-Issued ID Token=].
4. The [=Verifier=] resolves the client [=DID=] based on the value of the [=Self-Issued ID Token=] `sub` claim.
5. The [=DID Service=] returns the DID Document. The [=Verifier=] validates the [=Self-Issued ID Token=] following
   Section [[[#validating-self-issued-id-tokens]]].
6. The [=Verifier=] obtains the client's [=Credential Service=] endpoint address using the DID document as described in
   Section [[[#credential-service-endpoint-discovery]]]. The [=Verifier=] then issues a request with the access token
   to the [=Credential Service=] for a set of [=Verifiable Credentials=].
7. The [=Credential Service=] validates the access token and returns a Verifiable Presentation containg the requested
   credentials.

## Credential Service Endpoint Discovery

The client [=DID Service=] MUST make the [=Credential Service=] available as a `service` entry ([[[did-core]]], sect.
5.4) in the DID document that is resolved by its DID. The `type` attribute of the `service` entry MUST be
`CredentialService`.
The `serviceEndpoint` property MUST be interpreted by the Verifier as the base URL of the [=Credential Service=]. The
following is a non-normative example of a `Credential Service` entry:

```json
{
  "@context": [
    "https://www.w3.org/ns/did/v1",
    "https://w3id.org/dspace-dcp/v0.8"
  ],
  "service": [
    {
      "id": "did:example:123#identity-hub",
      "type": "CredentialService",
      "serviceEndpoint": "https://cs.example.com"
    }
  ]
}
```

## Credential Service Security

As described in the previous presentation flow, a [=Credential Service=] may require an access token when processing a
request from a [=Verifier=]. The format of the access token is not defined. How access control is defined in
a [=Credential Service=] is implementation-specific. For example, implementations may provide the ability to selectively
restrict access to resources.

### Submitting an Access Token

Implementations that support access control require an access token. To provide the opportunity
for [=Credential Service=] implementations to enforce proof-of-possession, the access token MUST be contained in the
`token`
claim of a Self-Issued ID Token as defined in Section [[[#self-issued-id-tokens]]]. The [=Self-Issued ID Token=] MUST
be submitted in the HTTP `Authorization` header prefixed with `Bearer` of the request.

## Resolution API

The Resolution API defines the [=Credential Service=] endpoint for querying credentials and returning a set
of [=Verifiable
Presentations=].

If a client is not authorized for an endpoint request, the [=Credential Service=] SHOULD return `4xx Client Error`. The
exact error code is implementation-specific.

### Query For Presentations

[=Verifiable Presentations=] can be queried by POSTing a `PresentationQueryMessage` message to the query endpoint:
`POST /presentations/query`.

The POST body is a `PresentationQueryMessage` JSON object with the following properties:

- `@context`: REQUIRED. Specifies a valid Json-Ld context ([[json-ld11]], sect. 3.1).
- `@type`: REQUIRED. A string specifying the `PresentationQueryMessage` type.
- `presentationDefinition`: OPTIONAL. A valid `Presentation Definition` according to
  the [Presentation Exchange Specification](https://identity.foundation/presentation-exchange/spec/v2.0.0/#presentation-definition).
- `scope`: OPTIONAL. An array of scopes corresponding to Section [[[#scopes]]].

A `PresentationQueryMessage` MUST contain either a `presentationDefinition` or a `scope` parameter. It is an error to
contain both.

The following are non-normative examples of the JSON body:

```json
{
  "@context": [
    "https://w3id.org/dspace-dcp/v0.8",
    "https://identity.foundation/presentation-exchange/submission/v1"
  ],
  "@type": "PresentationQueryMessage",
  "scope": []
}
```

```json
{
  "@context": [
    "https://w3id.org/dspace-dcp/v0.8",
    "https://identity.foundation/presentation-exchange/submission/v1"
  ],
  "@type": "PresentationQueryMessage",
  "presentationDefinition": "..."
}
```

#### Presentation Definitions

An implementations MAY support the `presentationDefinition` parameter. If it does not, it MUST return
`501 Not Implemented`. The `presentationDefinition` parameter contains a valid `Presentation Definition`
according to
the [Presentation Exchange Specification](https://identity.foundation/presentation-exchange/spec/v2.0.0/#presentation-definition).
The [=Credential Service=] will use the presentation definition to return a set of matching VPs in the format specified
by the definition.

#### Scopes

Implementations MAY support requesting presentation of [=Verifiable Credentials=] using a set of scope values.
A scope is an alias for a well-defined Presentation Definition. The specific scope value, and the
mapping between it and the respective Presentation Definition is out of scope of this specification.

A scope is a string value in the form:

`[alias]:[descriminator]`

The `[alias]` value MAY be implementation-specific.

##### The `org.eclipse.dspace.dcp.vc.type` Alias

The `vc.type` alias value MUST be supported and is used to specify access to a verifiable credential by type. For
example:

`org.eclipse.dspace.dcp.vc.type:Member`

denotes read-only access to the VC type `Member` and may be used to request a VC or VP.

##### The `org.eclipse.dspace.dcp.vc.id` Alias

The `org.eclipse.dspace.dcp.vc.id` alias value must be supported and is used to specify access to a verifiable
credential by id. For example:

`org.eclipse.dspace.dcp.vc.id:8247b87d-8d72-47e1-8128-9ce47e3d829d`

denotes read-only access to the VC identified by `8247b87d-8d72-47e1-8128-9ce47e3d829d` and may be used to request a
Verifiable Credential.

#### Query For Presentations Response

The response type of a presentation query is a `PresentationResponseMessage` with the following parameters:

- `@context`: REQUIRED. Specifies a valid Json-Ld context ([[json-ld11]], sect. 3.1).
- `@type`: REQUIRED. A string specifying the `PresentationResponseMessage` type.
- `presentation`: REQUIRED. An array of [=Verifiable Presentations=]. The [=Verifiable Presentations=] may be strings,
  JSON objects, or a combination of both depending on the format.

The following are non-normative examples of the JSON response body:

```json
{
  "@context": [
    "https://w3id.org/dspace-dcp/v0.8"
  ],
  "@type": "PresentationResponseMessage",
  "presentation": [
    "dsJdh...UMetV"
  ]
}
```

## Storage API

The Storage API defines the [=Credential Service=] endpoint for writing issued credentials, typically invoked by
an [=Issuer Service=].

If a client is not authorized for an endpoint request, the [=Credential Service=] SHOULD return `4xx Client Error`. The
exact error code is implementation-specific.

### Write Credentials

[=Verifiable Credentials=] can be written to the [=Credential Service=] by POSTing a `CredentialMessage` to the
`credentials` endpoint: `POST /credentials`.

If the POST is successful, credentials will be created and an HTTP `2XX` is returned.

The POST body is a `CredentialMessage` JSON object with the following properties:

- `@context`: REQUIRED. Specifies a valid Json-Ld context ([[json-ld11]], sect. 3.1).
- `@type`: REQUIRED. A string specifying the `CredentialMessage` type.
- `requestId`: REQUIRED. A string corresponding to the issuance request id.
- `credentials`: REQUIRED. An array of `CredentialContainer` Json objects corresponding to the schema
  specified in section [[[#the-credentialcontainer-object]]].

The following is a non-normative example of the JSON body:

```json
{
  "@context": [
    "https://w3id.org/dspace-dcp/v0.8"
  ],
  "@type": "CredentialMessage",
  "requestId": "...",
  "credentials": [
    {
      "@type": "CredentialContainer",
      "payload": ""
    }
  ]
}
```

#### The `CredentialContainer` Object

The `credentials` property contains an array of `CredentialContainer` objects. The `CredentialContainer` object contains
the following properties:

- `@type`: REQUIRED. A string specifying the `CredentialContainer` type.
- `payload`: REQUIRED. A [Json Literal]([[json-ld11]], sect. 4.2.2) containing a [=Verifiable Credential=] defined
  by ([[vc-data-model]]).