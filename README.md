# Team Exercise: Azure Cloud-hosted Data Processing Pipeline

The task is to build a cloud-hosted data processing pipeline in Azure. The pipeline will consume Json arrays, debatch those arrays into Json objects, validate each object, transform each object, and then run custom business rules on those objects before inserting them into a database.

The primary nonfunctional attributes that the client is looking for: Ease of maintenance, low cloud hosting costs, design simplicity, and a combination of scalability and performance (up to a point).

## Pipeline processing stages:

Listed out, the pipeline should execute the following functions:
1. Receive messages (Json arrays)
1. Debatch the messages into individual Json objects
1. Validate each object against a well-defined set of validation rules
1. Transform each object into the specified Json schema (shown below)
1. Run a second set of user-configured business rules, where the user-configured business rules are defined by a Json or XML configuration file
1. Insert the transformed Json object (from step #4) into a database
1. Insert all errors and warnings encountered during any of the transformation and/or validation steps into a database

The image below provides a visual depiction of the functional steps and ordering required to complete the exercise.

![Flow diagram showing the required steps for processing the data. Json arrays are first received from Azure Gen2 Storage and debatched into individual Json objects. Each Json object is validated, transformed, and then validated again. The transformed Json and all validation results must be stored somewhere.](images/team-exercise-01.png "Pipeline processing stages")

> The diagram and the steps do not imply that the team should implement any particular architecture. This diagram is merely showing the required steps to complete the exercise.

### Step 1 - Receive messages
For the purpose of this exercise, assume the Json arrays will first be inserted into Azure Data Lake Gen2 Storage. Your pipeline will need to pick up each Json array from that location. How you handle getting payloads from Azure Gen2 Storage into your pipeline is up to you.

### Step 2 - Debatch Json arrays into Json objects
Assume that what you receive will be a __Json array__. Each Json object in that array will look as follows, with only variations to the data (there will be no schema or data type changes):

```json
{
    "title": "The Price of Admiralty: The Evolution of Naval Warfare from Trafalgar to Midway",
    "format": "Paperback",
    "pages": 400,
    "isbn-13": "978-0140096507",
    "weight": 280.66,
    "language": "English",
    "authors": [
        {
            "order": 1,
            "first": "John",
            "last": "Keegan",
        },
    ],
    "publisher": "Pengiun Books",
    "edition": 3,
    "year": 1990,
    "month": 2,
    "dimensions": {
        "length": 18,
        "width": 13,
        "depth": 2.2
    },
    "rating": 4.56,
    "stock": 9801,
    "tags": [
        "Naval military history", "World War I", "World War II"
    ]
}
```
### Step 3 - Standard validation rules for each Json Object

The standard validation logic that must run for each Json object:

1. `pages` must be greater than 0. Otherwise, error.
1. `weight` must be greater than 0.0. Otherwise, error.
1. `authors` array must contain at least one item. Otherwise, error.
1. For each author in `authors`, the author shall have _either_ a first name or a last name. If no names are present, error. If one of the names is present, warning.
1. `year` must not exceed the current year. Otherwise, error.
1. `month` values must be between 1 and 12, inclusive. Otherwise, error.
1. `tags` array count must not exceed 20 items. Otherwise, warning.

An example of an error/warning Json payload is shown below, in this case for rule #1:

```json
{
    "severity": "Error",
    "code": "001",
    "message": "Pages must be greater than 0",
}
```

### Step 4 - Transform the Json object into a different format

The destination Json format for each Json object, per step #4:

```json
{
    "title": "The Price of Admiralty: The Evolution of Naval Warfare from Trafalgar to Midway",
    "format": "Paperback",
    "pages": 400,
    "isbn-13": "978-0140096507",
    "language": "English",
    "mainAuthor": "John Keegan",
    "publishInfo": "Pengiun Books, edition 3, Feb 1990",
    "dimensions": {
        "weight": 280.66,
        "length": 18,
        "width": 13,
        "depth": 2.2,
        "description": "18.0 cm x 13.0 cm x 2.2 cm (280.66 g)"
    },
    "rating": "4.6 out of 5 stars",
    "tagCount": 3
}
```

> The order of Json properties can vary from the example shown above.

### Step 5 - Apply user-defined business rules

The "user-defined business rules" are validation rules that end users supply. The user-defined business rules must be defined in a Json configuration file. These rules will be executed on the resulting, transformed Json that is output from __STEP 4__. These are _not_ applied on the original Json object.

Assume that end users can already build these rules on their own. You only need to supply a rules "engine" that consumes these rules and applies them to the output of Step 4. You can choose to build your own rules engine or reuse an existing rules engine. This rules engine does not need to distinguish between errors and warnings, unlike the previous set of rules.

For the purposes of this exercise, assume that your user-defined rules are as such:

1. `pages` shall be between 10 and 1000, inclusive.
1. `mainAuthor` shall be between 1 and 255 characters, inclusive.
1. `dimensions.weight` shall not exceed 11400
1. `dimensions.length` shall be between 0.1 and 100, inclusive.
1. `dimensions.length` shall not be greater than `dimensions.width`
1. `dimensions.depth` shall not be greater than `dimensions.width`
1. `pages` shall be greater than 100 when `weight` is greater than `200`
1. `rating` shall be in the format `{x} out of 5 stars` where {x} is a floating point number with one decimal point (e.g. 4.6)
1. `format` shall be either "Paperback" or "Hardcover"
1. `isbn-13` must be in the format `nnn-nnnnnnnnnn` where each "n" represents a single digit

## Constraints and assumptions
Your team is required to deal with the following constraints:

- Accurate cost calculation for running the pipeline (more on this below) [1]
- Minimal authentication/authorization boundary, such that pipeline components are not accessible to unauthenticated and unauthorized users
- 100% guaranteed message processing after messages are received by the pipeline from Azure Gen2 Storage. Regardless of whether business rules or validation rules fail, or if an Azure service fails, each message shall be tracked such that its failure or success can be investigated.
- All validation failures must be recorded in a database and queryable
- We will supply the Json array with 100,000 messages and a second set of 50,000 individual messages, and we will know in advance how many errors/warnings the pipeline should generate
- Assume you're building this solution for a client who has no IaaS, no existing infrastructure or IT assets, and is not part of a larger enterprise.
- Assume you may _only_ use FedRAMP moderate services in the Azure public cloud. (Note: This precludes the use of some SaaS products like MongoDB Atlas.)

Notably, you are otherwise free to choose how you implement the solution. Any Azure service is at your disposal and you may choose any programming language(s).

## Deliverables

- All diagrams explaining the data flow , logical acrhitecture , security architecture.
- Written rationale explaining why you made the architectural decisions you did. Include rationale for how you see your aproach could scale and be cost effective.

## Scoring Criteria

The exercise will be scored based on the following:

1. 25%: Cloud hosting cost to run the entire pipeline for a single day. To simulate this, assume that a 100,000 batch is received in the morning and 50,000 individual messages are received in the afternoon. [1]
1. 10%: Cloud hosting cost curve. That is, does processing higher volumes of batches (up to a max batch size of 1,000,000) increase costs linearly? If costs scale linearly or remain fixed, within a +/- 7% margin, full points are awarded. Otherwise, no points are awarded.
1. 20%: Process time for a 100,000 Json object array. The shorter the time, the higher the score. [2]
1. 10%: Lines of code. The fewer lines of code, the higher the score. [3]
1.  5%: Continuous integration/continuous delivery implementation (pass/fail)
1.  5%: Basic identity, authentication, and authorization implementation (pass/fail)
1.  5%: vNet integration (pass/fail)
1. 20%: Subjective assessment of the design [4]

[1] The cost measure shall include everything the pipeline uses to achieve the desired outcome, which is a user querying the database after all 150,000 items have been received, and assuming the pipeline must be in a 'ready' state when not processing data. This means the cost shall include (but is not limited to) all pipeline processing logic, any virtual machine costs, all data storage costs, and all API calls. Note the measurement period is __24 hours__. It is not the time it takes to run the pipeline from start to finish. A full score is awarded for a cost at or less than $30/day. A score of 0 is awarded if the job consumes more than $1000/day. A linear scoring gradient will determine the score for any cost between $30 and $1000.

[2] Scoring criteria: For all 100,000 objects, measure the time from when the pipeline is initiated to when the last Json object is inserted into the database. A full score is awarded if the job finishes in less than 5 minutes. A score of 0 is awarded if the job consumes more than 30 minutes. A linear scoring gradient will determine the score for any time between 5 and 30 minutes.

[3] Lines of code (LoC) are measured by properly formatted source code files. Follow all industry-standard conventions for formatting your code. All configuration files (excluding Azure Resource Manager Templates), Dockerfiles, and scripts (excluding CI/CD scripts) will count as "code" for the purpose of this scoring criteria. Any solution less than 300 total lines achieves a full score. Any solution that is greater than 5,000 lines of code achieves no score on this criterion. A linear scoring gradient will determine the score for any time between 5 and 30 minutes.

[4] Subjectively, we're looking for a design that's fast, scalable to a certain point, simple to learn and reason about, easy to modify, and easily maintained. If a team were to come in and take over this design, what would they need to know or learn?
