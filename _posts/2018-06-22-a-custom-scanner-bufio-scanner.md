---

layout: post
tags: ["Go"]
title: "Writing a custom bufio scanner function"

---

I recently had to work on a project that was originally a proof of concept i.e
converting to a full fledged application. In it's original form, there are no
tests, so I figured out that would be the first place I start from while
refactoring. While writing integration tests for `mysql`, I needed to be able to
setup the database by creating approipate schemas and all of that, ideally, I
would have used migrations that can be run easily during app startup and tests
initialization. But I didn't have the luxury here <sup>0</sup>, so I just moved
the entire sql dump to `testdata/init.sql`.

The problem now was how to import the file into the database (during CI, as I
wouldn't have access to the MySQL shell to run `source file`), I
reached out for `bufio#Scanner` to implement some parsing but figured out it would not serve my usecase as
it defaults to splitting content by lines. [See here](https://godoc.org/bufio/#NewScanner) . I needed to be able to distinguish sql statements.
A valid one for instance might span multiple lines and terminated by a
semi-colon (;). I then decided to write a custom splitting function..


Example sql is something like this



Here is what I ended up with for the splitting

```go

func TestDB_KeyExistsD(t *testing.T) {

	db, _ := sql.Open("user:passwd@tcp(localhost:3306)/test?parseTime=true")

	f, err := os.Open("testdata/init.sql")
	require.NoError(t, err)

	buf := bufio.NewScanner(f)

	buf.Split(func(data []byte, atEOF bool) (int, []byte, error) {

		// Trim out unneccessary whitespaces
		trimSpaces := func(b []byte) []byte {
			return bytes.TrimSpace(b)
		}

		if len(data) == 0 {
			return 0, nil, nil
		}

		// SQL statements are delimited by ';'
		if i := bytes.IndexByte(data, ';'); i >= 0 {
			return i + 1, trimSpaces(data[0:i]), nil
		}

		if atEOF {
			return len(data), trimSpaces(data), nil
		}

		return 0, nil, nil
	})

	for buf.Scan() {
		db.Exec(buf.Text())
	}

	// Real test comes here
}

``````

Hopefully this helps someone trying to implement something of this sort


#### Footnotes

<div id="footnotes"> </div>

[0] The major reason why I couldn't use migrations was because the app uses an
existing database..If I have made the dump the first migration, I risk having
errors on running it against the production database as indexes that already
exist would refuse to be created again and thus the app would refuse to start
