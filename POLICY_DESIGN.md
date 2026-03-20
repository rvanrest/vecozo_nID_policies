# Policy design

Om de kwaliteit van de policies te waarborgen, zijn er een aantal design regels opgesteld. Deze
regels zijn opgesteld om de code leesbaar en consistent te houden, maar ook om het gebruik van de
policies te vereenvoudigen.

<!--toc-->

- [Policy design](#policy-design)
  - [Resultaat](#resultaat)
    - [Allowed](#allowed)
    - [HTTP status](#http-status)
    - [Body](#body)
  - [Functionele foutmeldingen](#functionele-foutmeldingen)
    - [Deny rules](#deny-rules)
    - [Error structuur](#error-structuur)

<!-- tocstop -->

## Resultaat

Omdat we gebruik maken van verschillende variabelen en hulpmiddelen om de policies te schrijven, is
het belangrijk om te weten wat het resultaat is van een policy. Het resultaat van de policy moet een
object zijn genaamd `result`. Dit object moet de volgende variabelen bevatten:

| Variabele     | Beschrijving                                                   |
| ------------- | -------------------------------------------------------------- |
| `allowed`     | Of het request is toegestaan of niet.                          |
| `http_status` | De HTTP status code die wordt teruggegeven aan de gebruiker.   |
| `body`        | De HTTP response body die wordt teruggegeven aan de gebruiker. |

Dit maakt het mogelijk om bij de evaluatie van de policy alleen de nodige output te krijgen en niet
onnodige variabelen.

Het resultaat kan makkelijk opgebouwd worden met andere variabelen zoals in de `example.rego`:

```rego
default result["allowed"] := false

result["body"] := {"errors": deny}

result["allowed"] if {
    count(deny) == 0
}

result["status"] := 500
```

### Allowed

OPA policies worden gebruikt om te bepalen of een request is toegestaan of niet. Om dit te doen,
wordt er gebruik gemaakt van de `allowed` variabele. Deze variabele moet altijd worden gebruikt om
aan te geven of een request is toegestaan of niet. Omdat deze variabele altijd aanwezig moet zijn is
het een goed idee om een standaard waarde te definiëren. Dit kan op de volgende manier:

```rego
default allowed := false
```

### HTTP status

Door een HTTP status code terug te geven in het resultaat krijgt de gebruiker een goed beeld van
waarom een request is afgekeurd. Het is daarom een goed idee om een standaard waarde te definiëren
voor de `http_status`. Dit kan op de volgende manier:

```rego
# 200 is de status code van OK
default http_status := 200
```

Wanneer er een andere status code nodig is, kan deze worden aangepast. Dit kan op de volgende
manier:

```rego
http_status := 401 if {
    not is_token_valid
}
```

### Body

De response body is de belangrijkste informatie voor de gebruiker. Hierin staat namelijk de
informatie waarom een request is afgekeurd. De in de body houden we de graphql response body
conventie aan (dit is het makkelijkst voor de gebruikers), dit ziet eruit als volgd:

```json
{
  "errors": [
    {
      "message": "A where for 'instelling' is required and must be set to 123456.",
      "extensions": {
        "code": "BAD_REQUEST"
      }
    }
  ]
}
```

## Functionele foutmeldingen

We gebruiken de body om functionele foutmeldingen terug te geven aan de gebruiker. Deny rules zijn
hierbij de makkelijkste manier om zo veel mogelijk informatie terug te geven aan de gebruiker.

### Deny rules

We op met behulp van `deny` rules functionele foutmeldingen op. Elke `deny` rule bekijkt een
specifiek gedeelte van het binnenkomende request en bepaald of dit toegestaan is of niet. Wanneer
het request niet voldoet aan de regels, wordt er een error message toegevoegd aan de `deny` array.
Deze array wordt vervolgens teruggegeven aan de gebruiker.

Een `deny` rule ziet eruit als volgt:

```rego
deny contains msg if {
    some i, j
    query_ast.Operations[i].SelectionSet[j].Alias == "bemiddeling"
    not where_instelling_required(query_ast.Operations[i].SelectionSet[j], "123456")
    msg := {
        "message": err("A where for 'instelling' is required and must be set to 123456.", i),
        "extensions": {"code": errors.INTERNAL_SERVER_ERROR},
    }
}
```

Allereerst wordt er bekeken of deze regel opgaat voor de het huidige request. Deze regel is alleen
van toepassing voor de "bemiddeling" query.

Daarna worden de regels geëvalueerd. Dit wordt gedaan vanuit het negatieve. Dit betekend dat we
kijken of het request niet voldoet aan de regels. Wanneer het request niet voldoet aan de regels,
wordt er een error gegeven aan `msg` wat de error messages opbouwt.

### Error structuur

Een error heeft een aantal vaste onderdelen. Deze onderdelen zijn:

| Variabele    | Beschrijving                                               |
| ------------ | ---------------------------------------------------------- |
| `message`    | De error message die wordt teruggegeven aan de gebruiker.  |
| `extensions` | De error extensions, extra informatie over het type error. |

De `message` is een string die wordt teruggegeven aan de gebruiker. Deze string kan worden opgebouwd
met behulp van de `err` functie, hiermee wordt de error mooi opgebouwd. Een error ziet er als volgt
uit:

```rego
    msg := {
        "message": err("A where for 'instelling' is required and must be set to 123456.", i),
        "extensions": {"code": errors.INTERNAL_SERVER_ERROR},
    }
```
