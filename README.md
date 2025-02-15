# Building a RESTful API with Go and MongoDB

In this tutorial, we'll walk through how to build a simple RESTful API using Go and MongoDB. The API will manage a store of movies with the following functionalities:

- **Create** a new movie
- **Retrieve** a list of movies
- **Find** a specific movie by ID
- **Update** an existing movie
- **Delete** a movie

---

## 1. Fetching Dependencies

To get started, install the necessary dependencies:

```bash
go get github.com/BurntSushi/toml gopkg.in/mgo.v2 github.com/gorilla/mux
```

**Dependency Overview:**
- **toml**: Parse configuration files for MongoDB credentials.
- **mux**: HTTP request router and dispatcher.
- **mgo**: MongoDB driver for Go.

---

## 2. API Structure

Create a file named `app.go` and include the following code:

```go
package main

import (
	"net/http"
	"github.com/gorilla/mux"
	"log"
)

func main() {
	r := mux.NewRouter()
	r.HandleFunc("/movies", GetMoviesEndpoint).Methods("GET")
	r.HandleFunc("/movies/{id}", GetMovieEndpoint).Methods("GET")
	r.HandleFunc("/movies", CreateMovieEndpoint).Methods("POST")
	r.HandleFunc("/movies/{id}", UpdateMovieEndpoint).Methods("PUT")
	r.HandleFunc("/movies/{id}", DeleteMovieEndpoint).Methods("DELETE")

	log.Fatal(http.ListenAndServe(":3000", r))
}
```

Run the server locally:

```bash
go run app.go
```

Access the server at: `http://localhost:3000/movies`.

---

## 3. Defining the Movie Model

Create a `movie.go` file with the following structure:

```go
package main

type Movie struct {
	ID          string `json:"id" bson:"_id"`
	Name        string `json:"name"`
	CoverImage  string `json:"cover_image"`
	Description string `json:"description"`
}
```

---

## 4. Database Access Layer

### 4.1 Establish Database Connection

```go
package main

import (
	"gopkg.in/mgo.v2"
	"log"
)

var db *mgo.Database

const (
	HOST       = "localhost:27017"
	DATABASE   = "movie_db"
	COLLECTION = "movies"
)

func connect() {
	session, err := mgo.Dial(HOST)
	if err != nil {
		log.Fatal(err)
	}
	db = session.DB(DATABASE)
}
```

### 4.2 Database Operations

```go
func FindAll() ([]Movie, error) {
	var movies []Movie
	err := db.C(COLLECTION).Find(nil).All(&movies)
	return movies, err
}

func FindByID(id string) (Movie, error) {
	var movie Movie
	err := db.C(COLLECTION).FindId(id).One(&movie)
	return movie, err
}

func Insert(movie Movie) error {
	return db.C(COLLECTION).Insert(&movie)
}

func Update(movie Movie) error {
	return db.C(COLLECTION).UpdateId(movie.ID, &movie)
}

func Delete(id string) error {
	return db.C(COLLECTION).RemoveId(id)
}
```

---

## 5. API Endpoints

### 5.1 Create a Movie

```go
func CreateMovieEndpoint(w http.ResponseWriter, r *http.Request) {
	var movie Movie
	json.NewDecoder(r.Body).Decode(&movie)
	movie.ID = bson.NewObjectId().Hex()
	Insert(movie)
	json.NewEncoder(w).Encode(movie)
}
```

**Test with cURL:**
```bash
curl -X POST -H "Content-Type: application/json" -d '{"name":"Inception","cover_image":"https://example.com/inception.jpg","description":"Sci-fi thriller"}' http://localhost:3000/movies
```

---

### 5.2 Retrieve All Movies

```go
func GetMoviesEndpoint(w http.ResponseWriter, r *http.Request) {
	movies, err := FindAll()
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}
	json.NewEncoder(w).Encode(movies)
}
```

**Test with cURL:**
```bash
curl -X GET http://localhost:3000/movies
```

---

### 5.3 Retrieve a Movie by ID

```go
func GetMovieEndpoint(w http.ResponseWriter, r *http.Request) {
	params := mux.Vars(r)
	movie, err := FindByID(params["id"])
	if err != nil {
		http.Error(w, "Movie not found", http.StatusNotFound)
		return
	}
	json.NewEncoder(w).Encode(movie)
}
```

**Test with cURL:**
```bash
curl -X GET http://localhost:3000/movies/{id}
```

---

### 5.4 Update a Movie

```go
func UpdateMovieEndpoint(w http.ResponseWriter, r *http.Request) {
	params := mux.Vars(r)
	var movie Movie
	json.NewDecoder(r.Body).Decode(&movie)
	movie.ID = params["id"]
	Update(movie)
	json.NewEncoder(w).Encode(movie)
}
```

**Test with cURL:**
```bash
curl -X PUT -H "Content-Type: application/json" -d '{"name":"Inception Updated","cover_image":"https://example.com/inception.jpg","description":"Sci-fi thriller updated"}' http://localhost:3000/movies/{id}
```

---

### 5.5 Delete a Movie

```go
func DeleteMovieEndpoint(w http.ResponseWriter, r *http.Request) {
	params := mux.Vars(r)
	Delete(params["id"])
	json.NewEncoder(w).Encode(map[string]string{"result": "success"})
}
```

**Test with cURL:**
```bash
curl -X DELETE http://localhost:3000/movies/{id}
```

---

## Next Steps

To enhance your API, consider:

- Writing unit tests for each endpoint.
- Building a user-friendly front-end.
- Deploying the API to a cloud platform like AWS.
- Implementing security best practices like input validation and authentication.


