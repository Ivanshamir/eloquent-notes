1.**n-gram:** In the fields of computational linguistics and probability, an n-gram (sometimes also called Q-gram) is a contiguous sequence of n items from a given sample of text or speech. The items can be phonemes, syllables, letters, words or base pairs according to the application. The n-grams typically are collected from a text or speech corpus. When the items are words, n-grams may also be called shingles. Using Latin numerical prefixes, an n-gram of size 1 is referred to as a "unigram"; size 2 is a "bigram" (or, less commonly, a "digram"); size 3 is a "trigram". English cardinal numbers are sometimes used, e.g., "four-gram", "five-gram", and so on. In computational biology, a polymer or oligomer of a known size is called a k-mer instead of an n-gram, with specific names using Greek numerical prefixes such as "monomer", "dimer", "trimer", "tetramer", "pentamer", etc., or English cardinal numbers, "one-mer", "two-mer", "three-mer", etc.

### Applications
An n-gram model is a type of probabilistic language model for predicting the next item in such a sequence in the form of a (n − 1)–order Markov model. n-gram models are now widely used in probability, communication theory, computational linguistics (for instance, statistical natural language processing), computational biology (for instance, biological sequence analysis), and data compression. Two benefits of n-gram models (and algorithms that use them) are simplicity and scalability – with larger n, a model can store more context with a well-understood space–time tradeoff, enabling small experiments to scale up efficiently.

2. **MySQL ngram Full Text Parser** The built-in MySQL full-text parser limits the opening and finish of words by means of white space. When it comes to other languages such as Chinese, Japanese, or Korean, etc., this is a restriction for the reason that these languages do not use word delimiters.
To report this issue, MySQL providing the ngram full-text parser. Later version 5.7.6, MySQL comprised ngram full-text parser as a built-in server plugin, sense that MySQL loads this plugin routinely when the MySQL database server starts. MySQL provisions ngram full-text parser for equally InnoDB and MyISAM storing engines.
A ngram is a connecting sequence of a number of characters from an order of text. The main function of ngram full-text parser is tokenizing a order of text into a connecting order of n characters.
```
n=1: 'a', 'b', 'c', 'd'
n=2: 'ab', 'bc', 'cd'
n=3: 'abc', 'bcd'
n=4: 'abcd'
```
The ngram full-text parser is a built-in server plugin. As with other built-in server plugins, it is automatically loaded when the server is started.
The full-text search syntax described in Section 12.10, “Full-Text Search Functions” applies to the ngram parser plugin. Differences in parsing behavior are described in this section. Full-text-related configuration options, except for minimum and maximum word length options (innodb_ft_min_token_size, innodb_ft_max_token_size, ft_min_word_len, ft_max_word_len) are also applicable.To add a FULLTEXT index to an existing table, you can use ALTER TABLE or CREATE INDEX. For example:
```
CREATE TABLE articles (
      id INT UNSIGNED AUTO_INCREMENT NOT NULL PRIMARY KEY,
      title VARCHAR(200),
      body TEXT
     ) ENGINE=InnoDB CHARACTER SET utf8;

ALTER TABLE articles ADD FULLTEXT INDEX ft_index (title,body) WITH PARSER ngram;

# Or:

CREATE FULLTEXT INDEX ft_index ON articles (title,body) WITH PARSER ngram;
```
ngram Parser Space Handling
The ngram parser eliminates spaces when parsing. For example:
“ab cd” is parsed to “ab”, “cd”
“a bc” is parsed to “bc”

3. **POINTS** :
A Point consists of X and Y coordinates, which may be obtained using the ST_X() and ST_Y() functions, respectively. These functions also permit an optional second argument that specifies an X or Y coordinate value, in which case the function result is the Point object from the first argument with the appropriate coordinate modified to be equal to the second argument.
For Point objects that have a geographic spatial reference system (SRS), the longitude and latitude may be obtained using the ST_Longitude() and ST_Latitude() functions, respectively. These functions also permit an optional second argument that specifies a longitude or latitude value, in which case the function result is the Point object from the first argument with the longitude or latitude modified to be equal to the second argument.

Unless otherwise specified, functions in this section handle their geometry arguments as follows:
If any argument is NULL, the return value is NULL.
If any geometry argument is a valid geometry but not a Point object, an ER_UNEXPECTED_GEOMETRY_TYPE error occurs.
If any geometry argument is not a syntactically well-formed geometry, an ER_GIS_INVALID_DATA error occurs.
If any geometry argument is a syntactically well-formed geometry in an undefined spatial reference system (SRS), an ER_SRS_NOT_FOUND error occurs.
If an X or Y coordinate argument is provided and the value is -inf, +inf, or NaN, an ER_DATA_OUT_OF_RANGE error occurs.
If a longitude or latitude value is out of range, an error occurs:
If a longitude value is not in the range (−180, 180], an ER_LONGITUDE_OUT_OF_RANGE error occurs.
If a latitude value is not in the range [−90, 90], an ER_LATITUDE_OUT_OF_RANGE error occurs.
Ranges shown are in degrees. The exact range limits deviate slightly due to floating-point arithmetic.
Otherwise, the return value is non-NULL.
4. 
