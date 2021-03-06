# Analysis of Security-related Code on Stack Overflow

This is the repository for producing the results of our paper published in [The 41st ACM/IEEE International Conference on Software Engineering (ICSE 2019)](https://2019.icse-conferences.org/track/icse-2019-Technical-Papers#event-overview).

The title of the paper is: How Reliable is the Crowdsourced Knowledge of Security Implementation? [[pdf]](https://arxiv.org/pdf/1901.01327.pdf)
## Dependencies

- PostgreSQL
- Neo4j
- stackexchange-dump-to-postgres
- Java8
- java-baker
- CCFinder
- python3
- psycopg2
- numpy
- scipy
- pandas
- neo4j python driver
- matplotlib


## Usage

### Dataset

Obtain Stack Overflow data dump from [https://archive.org/details/stackexchange](https://archive.org/details/stackexchange). Then use stackexchange-dump-to-postgres to dump the data from xml files to PostgreSQL database.

There are roughly about 47 million posts in the 10 year period from 2008 to 2017, including all topics, such as not Java language and not security related posts. The next step is to filter out posts that contain Java security related code snippets.

### Security related code extraction

Make sure you have JDK 1.8, and the PostgreSQL server and Neo4j server is running. Then use the following command with the range of post id to start extraction.

```bash
./gradlew run -PappArgs="['39800000', '40000000']"  # parse postid >= 39800000 AND postid < 40000000
```

To speed up the extraction process, many extraction processes can be run in parallel for disjoint post id ranges. These processes can be distributed to different compute nodes within a computer cluster as long as they can access the database.

The extracted snippets are saved in the following table

```SQL
CREATE TABLE snippets(
    id serial,
    code VARCHAR(24060),
    postid INT,
    PRIMARY KEY (id),
    FOREIGN KEY (ID) REFERENCES posts(id)
 );
```

With the extracted code snippets inserted using:

```SQL
INSERT INTO snippets(code, indx, postid)
VALUES($aesc6$"+code+"$aesc6$,"+indx+","+postid+")
ON CONFLICT DO NOTHING;
```

Use the following SQL to get unprocessed question posts, for example, within the first 2 million posts:

```SQL
SELECT posts.id, posts.parentid, posts.body, posts.tags
FROM posts INNER JOIN marks ON posts.id=marks.postid
WHERE posts.id < 2000000 AND posts.parentid IS NULL
AND (marks.processed IS NULL OR marks.processed=FALSE);
```


### Clone detection

Use the detector python module ([py/detector.py](py/detector.py)) for clone detection and save the results to PostgreSQL database.

``` python
from detector import *
detect_clones(snippet_dname, ccf_opt, snippet_id_range, connect_str)
```

Manually running CCFinder can be found in CCFinder's documentation. Below are some common usages.
To start clone detection, in directory `ccfinderx-core/ccfx` run:

```bash
ccfx d java -dn ~/code/nbooks/snippets -o ~/code/nbooks/allsnippets.ccfxd
# or with options
ccfx d java -b 30 -dn ~/code/nbooks/snippets -o ~/code/nbooks/allsnippets_b30.ccfxd
```

To view the detection result, run:
```bash
ccfx p allsnippets.ccfxd
```

Use the following commands to export clone metrics and file metrics

```bash
ccfx m allsnippets.ccfxd -c -o clonemetrics.tsv -f -o filemetrics.tsv
```

### Analysis

All the analysis are included in the python module [py](py).

    - dbinfo.py
      contains database host name, user name, password. Input according to you setting, then change dbinfo_template.py to dbinfo.py.

    - ui.py
      provides the labeling user interface with syntax highlight. Labeling results are saved to database in labels table.

    - labeled.py
      provides functions to query all the informations (eg. scores, reputations) related to labeled posts.

    - clones.py
      provides classes to abstract the clone groups and clone instances.

    - cgraph.py
      provide functions to store clone relations to Neo4j graph database, and execute equivalent but simpler query of labeled posts.

    - statstest.py
      provides statistical testing on different aspects of secure vs insecure posts.

    - plotter.py
      contains all the plotting functions, including style custimization.
