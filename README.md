# BigQuery SQL Driver

[![BigQuery database/sql driver in Go.](https://goreportcard.com/badge/github.com/flarco/bigquery)](https://goreportcard.com/report/github.com/flarco/bigquery)
[![GoDoc](https://godoc.org/github.com/flarco/bigquery?status.svg)](https://godoc.org/github.com/flarco/bigquery)

This library is compatible with Go 1.17+

Please refer to [`CHANGELOG.md`](CHANGELOG.md) if you encounter breaking changes.

- [DSN](#dsn-data-source-name)
- [Usage](#usage)
- [Benchmark](#benchmark)
- [License](#license)
- [Credits and Acknowledgements](#credits-and-acknowledgements)

This library provides fast implementation  of the BigQuery Client as a database/sql drvier.

#### DSN Data Source Name

The BigQuery driver accepts the following DSN    

 * 'bigquery://projectID/[location/]datasetID?queryString'
 
    Where queryString can optionally configure the following option:
      - credentialsFile
      - credentialJSON (with url-base64 encoded json)
      - endpoint
      - userAgent
      - apiKey
      - quotaProject
      - scopes
  
 
Since this library uses [Google Cloud API](google.golang.org/api/bigquery/v2) 
you can pass your credentials via GOOGLE_APPLICATION_CREDENTIALS environment variable.

## Usage:


```go
package main

import (
    "database/sql"
    "fmt"
    "context"
    "log"
   _ "github.com/flarco/bigquery"
)


type Participant struct {
	Name string
	Splits []float64
}


func main() {

	db, err := sql.Open("bigquery", "bigquery://myProjectID/mydatasetID")
	if err != nil {
		log.Fatal(err)
	}
	defer db.Close()

	SQL := `WITH races AS (
		  SELECT "800M" AS race,
		    [STRUCT("Ben" as name, [23.4, 26.3] as splits), 
		 	 STRUCT("Frank" as name, [23.4, 26.3] as splits)
			]
		       AS participants)
		SELECT
		  race,
		  participant
		FROM races r
		CROSS JOIN UNNEST(r.participants) as participant`

	rows, err := db.QueryContext(context.Background(), SQL)
	if err != nil {
		log.Fatal(err)
	}
	defer rows.Close()
	for rows.Next() {
		if err := rows.Err(); err != nil {
			log.Fatal(err)
		}
		var race string
		var participant Participant
		err = rows.Scan(&race, &participant)
		fmt.Printf("fetched: %v %+v\n", race, participant)
		if err != nil {
			log.Fatal(err)
		}
	}
}
```

## Benchmark

Benchmark runs 3 times the following queries:

- primitive types:

```sql 
   SELECT state,gender,year,name,number 
   FROM `bigquery-public-data.usa_names.usa_1910_2013` LIMIT 10000
```

- complex type (repeated string/repeated record)

```sql 
   SELECT  t.publication_number, t.inventor, t.assignee, t.description_localized 
   FROM `patents-public-data.patents.publications_202105` t ORDER BY 1 LIMIT 1000
```

In both case database/sql driver is faster and allocate way less memory than [GCP BigQuery API Client](https://cloud.google.com/bigquery/docs/reference/libraries#client-libraries-install-go)

```go
goos: darwin
goarch: amd64
pkg: github.com/flarco/bigquery/bench
cpu: Intel(R) Core(TM) i9-9980HK CPU @ 2.40GHz
Benchmark_Primitive_GCPClient
database/gcp: primitive types 3.918388531s
Benchmark_Primitive_GCPClient-16    	       1	3918442974 ns/op	42145144 B/op	  830647 allocs/op
Benchmark_Primitive_SQLDriver
database/sql: primitive types 2.880091637s
Benchmark_Primitive_SQLDriver-16    	       1	2880149491 ns/op	22301848 B/op	  334547 allocs/op
Benchmark_Complex_GCPClient
database/gcp: structured types 3.303497894s
Benchmark_Complex_GCPClient-16      	       1	3303548761 ns/op	11551216 B/op	  154660 allocs/op
Benchmark_Complex_SQLDriver
database/sql: structured types 2.690012577s
Benchmark_Complex_SQLDriver-16      	       1	2690056675 ns/op	 6643176 B/op	   71562 allocs/op
PASS
```

## License

The source code is made available under the terms of the Apache License, Version 2, as stated in the file `LICENSE`.

Individual files may be made available under their own specific license,
all compatible with Apache License, Version 2. Please see individual files for details.

##  Credits and Acknowledgements

**Library Author:** Adrian Witas
