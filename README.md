# bitfield standard protocols

## Setting Up Your Environment

Contributing to the community standards repository requires proper authentication and tool configuration. The process involves creating GitHub credentials and configuring R to interact with the repository seamlessly.

First, establish your GitHub presence by creating an account if you don't have one. Generate a Personal Access Token (PAT) with appropriate permissions to read and write to repositories with the code shown below (study the respective function documentation, as this is outside the scope of the bitfield package and may change in the future). The "repo" scope provides full repository access, while "public_repo" scope limits access to public repositories only.

```r
# set up GitHub token (opens browser)
usethis::create_github_token()

# store token securely in R
gitcreds::gitcreds_set()

# verify token works
bf_standards(action = "list")                        # should return repository contents
```

The `usethis::create_github_token()` function opens your browser to GitHub's token creation interface with appropriate scopes pre-selected. Copy the generated token and paste it when prompted by `gitcreds::gitcreds_set()`. This stores the token securely in your system's credential manager, making it available for future R sessions without exposing it in your code. Furthermore, configure your Git identity to ensure proper attribution for contributions.

```r
# set Git configuration
usethis::use_git_config(user.name = "Your Name", user.email = "your.email@example.com")
```

 

## Develop a New Protocol

Creating effective bitfield protocols requires balancing information content with bit efficiency while ensuring broad applicability across research contexts. The protocol design process follows several key principles rooted in information theory and practical usability.

Begin by identifying the essential information that must be preserved when data moves between workflows. Focus on one atomic concept per protocol, maximizing flexibility and reusability. Consider which downstream applications might use your data and what information they would need to make informed decisions.

Let's walk through the process of developing a soil moisture confidence protocol. In this example, you want to enhance soil moisture measurements by adding a confidence metric and change a previously existing protocol that encodes the soil moisture values themselves with 6 bits currently to 7 bits, for higher accuracy. First, analyze the data characteristics and derive additional data items that shall be encoded.

```r
moisture_data <- c(15.2, 28.7, 42.1, 8.9, 35.6)      # soil moisture percentages
sensor_age <- c(2, 0.5, 3, 1, 2.5)                   # sensor age in years  
calibration_days <- c(30, 5, 180, 15, 90)            # days since last calibration

# determine confidence based on sensor reliability
confidence_scores <- pmax(0, 100 - (sensor_age * 10) - (calibration_days * 0.5))
range(confidence_scores)                             # 0 to 95
```

Then, design your encoding strategy by analyzing the data range and required precision. Here, you'd need 7 bits to encode confidence scores from 0-100 with sufficient granularity because you'd want to encode at least 1% steps. You'd also want to encode the actual soil-moisture values in 1% steps, so allocate 7 bits to this as well (see the best practices below for a more detailed explanation on how this example works).

```r
# new protocol to record soil moisture confidence
moisture_confidence <- bf_protocol(
  name = "soil_moisture_confidence",
  description = "Confidence calculated from {sensorAge}-year sensor and {sinceCalibration} days since calibration, encoded as 7-bit integer (0-127) where 127 = 100% confidence",
  test = function(sensorAge, sinceCalibration) {
    # calculate confidence score (0-100)
    confidence <- pmax(0, 100 - (sensorAge * 10) - (sinceCalibration * 0.5))
    # scale to 0-127 for 7-bit encoding
    round(confidence * 1.27)
  },
  example = list(
    sensorAge = c(1, 2.5, 0.5),
    sinceCalibration = c(15, 90, 5)
  ),
  type = "int",
  # allow 128 confidence levels (0-127)
  bits = 7  
)

# enhanced protocol that allows storing values with greater precision.
soil_moisture <- bf_protocol(
  name = "soil_moisture_percent", 
  description = "Soil moisture {valMoisture}% encoded as 7-bit integer (0-127) with 0.8% precision, where 127 = 100% field capacity",
  test = function(valMoisture) {
    # encode 0-100% moisture in 7 bits with 0.8% precision
    pmin(round(valMoisture * 1.27), 127)
  },
  example = list(valMoisture = c(15.2, 28.7, 42.1)),
  type = "int", 
  bits = 7,
  version = "1.1.0",
  extends = "soil_moisture_1.0.0",
  note = "New sensors allow greater precision and we thus need more bits to represent the values."
)
```

This atomic approach allows users to combine protocols flexibly, they might use moisture confidence with temperature measurements, or soil moisture percentages with different quality metrics, depending on their specific workflow needs.

## Interacting with the Standards Repository

The `bf_standards()` function provides a streamlined interface for discovering, retrieving, and contributing protocols. The function handles the complexity of GitHub API interactions while maintaining version control and proper attribution.

Start by exploring the repository structure to understand existing protocols and identify gaps your contribution might fill.

```r
# list all available protocols
all_protocols <- bf_standards(action = "list")

# filter by domain
ecological_protocols <- all_protocols[grepl("environmental/", all_protocols)]
```

Domains do not exist at the publication of this manuscript, they shall grow organically from community interaction. Pull protocols to examine their structure, extend their functionality, or use them as templates for new developments.

```r
# pull a specific protocol (that shall be enhanced)
existing_protocol <- bf_standards(
  protocol = "soil_moisture",
  remote = "environmental/soil",
  action = "pull"
)

# examine the protocol structure
str(existing_protocol)
```

Push your new, tested protocol to the repository, specifying the appropriate domain and subdomain for organization.

```r
# Push new protocol
bf_standards(
  protocol = moisture_confidence,
  remote = "environmental/soil",                     # choose appropriate subdirectory
  action = "push",
  version = "1.0.0",
  change = "Initial Soil moisture confidence."
)
```

When enhancing existing protocols, increment version numbers appropriately (see best practices below) and document changes clearly.

```r
# push update
bf_standards(
  protocol = soil_moisture,
  remote = "environmental/soil",
  action = "push",
  version = "1.1.0", 
  change = "Encode soil moisture values with 7 intead of the previous 6 bits."
)
```

The repository automatically archives previous versions, maintaining reproducibility while enabling protocol evolution. Each contribution receives proper attribution and generates citation-ready metadata.

## Best Practices

Effective protocol development requires attention to both technical implementation and community collaboration. Follow these guidelines to maximize the impact and longevity of your contributions.

**1. Design Atomic Protocols**

Each protocol should encode exactly one concept or measurement, but can have multiple, related inputs (it must be mappable to the encoding types `bool`, `enum`, `int` and `float`). However, resist the temptation to combine multiple related measurements into a single protocol, as this reduces flexibility and reusability.

```r
# good: Atomic protocol - single concept (confidence)
moisture_confidence <- bf_protocol(
  name = "soil_moisture_confidence",
  test = function(sensorAge, sinceCalibration) {
    confidence <- pmax(0, 100 - (sensorAge * 10) - (sinceCalibration * 0.5))
    round(confidence * 1.27)
  },
  type = "int"
)

# avoid: compound protocol combining multiple concepts
bf_protocol(
  name = "complete_soil_assessment", 
  test = function(moisture, temperature, ph, confidence) {
    # don't return multiple values - creates inflexible monolithic flags
    list(moist = moisture, temp = temperature, ph = ph, conf = confidence)  # Won't encode!
  }
)

# avoid: overly specific implementations
bf_protocol(
  name = "plot_7_moisture_june_2023_sensor_A",       # too specific
  test = function(moisture_reading) { moisture_reading }
)
```

Atomic protocols can be combined flexibly in registries, allowing users to select only the components relevant to their workflow while maintaining the ability to extend with additional protocols as needed. A user studying soil hydrology might combine `soil_moisture_percent` with `drainage_rate`, while someone focused on sensor networks might use `soil_moisture_confidence` with `battery_status`, the same moisture measurement protocol serves both workflows effectively.

**2. Documentation**

Use the `glue` syntax in descriptions to create systematic documentation that explains both input values and bit encoding. Parameter names in `{braces}` get replaced in the legend with actual values. This systematic approach to descriptions ensures that when users decode bitfields, they understand not just the input values but also how those values were transformed into bit representations. The description "Soil moisture 25.4% encoded as 7-bit integer (0-127) with 0.8% precision" immediately tells users both the original value and the encoding scheme used.

```r
moisture_confidence <- bf_protocol(
  description = "Confidence calculated from {sensorAge}-year sensor and {sinceCalibration} days since calibration, encoded as 7-bit integer (0-127) where 127 = 100% confidence",
  # ... rest of protocol
)

soil_moisture <- bf_protocol(
  description = "Soil moisture {valMoisture}% encoded as 7-bit integer (0-127) with 0.8% precision, where 127 = 100% field capacity",
  # ... rest of protocol
)
```

**3. Quality Control**

Always test your protocols with challenging datasets to identify potential failures before sharing.

```r
# create test data that stresses your protocol, revealing potential encoding failures before community use
test_data <- data.frame(
  moisture = c(0, 15.2, 50.0, 100.0, NA),            # boundary cases: dry, wet, missing
  sensor_age = c(0, 1, 3, 10, 2),                    # include extreme age (10 years)
  days_cal = c(0, 30, 90, 1000, 60)                  # include severely overdue (1000 days)
)

# test both protocols with edge cases
test_registry <- bf_registry(name = "test", description = "testing soil protocols")

# specify na.val for robust handling of missing data
test_registry <- bf_map(protocol = moisture_confidence, data = test_data,
                       sensorAge = sensor_age, sinceCalibration = days_cal,
                       registry = test_registry, na.val = 0L)

test_registry <- bf_map(protocol = soil_moisture, data = test_data,
                       valMoisture = moisture, registry = test_registry, na.val = 0L)

# verify the full encode/decode cycle works correctly
field <- bf_encode(registry = test_registry)
decoded <- bf_decode(field, registry = test_registry, verbose = FALSE)
# -> check that extreme values are handled as expected
```

**4. Version Management**

Use semantic versioning consistently. Increment major versions for breaking changes, minor versions for new functionality, and patch versions for bug fixes. Document changes meaningfully.

```r
# version update example
bf_standards(
  protocol = "my_fix",
  version = "2.1.3"                                  # major.minor.patch
  change = "Fixed edge case handling for zero-diversity samples (patch)"
  # vs
  change = "Added habitat quality weighting parameter (minor)" 
  # vs 
  change = "Changed output format from list to vector (major)",
  action = "push"
)
```

Contributors receive academic recognition through automatic DataCite-compatible metadata generation. To obtain actual DOIs for your protocols, upload your complete registry to Zenodo and associate it with the 'Bitfield Community Standards' group ([https://www.zenodo.org/communities/bitfield-standards](https://zenodo.org/communities/bitfield-standards)), making your contributions discoverable and permanently citable.
