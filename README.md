# FealtyX

package main

import (
    "bytes"
    "encoding/json"
    "errors"
    "fmt"
    "io/ioutil"
    "log"
    "math/rand"
    "net/http"
    "strconv"
    "sync"
)

// Student struct represents a student's information
type Student struct {
    ID    int    `json:"id"`
    Name  string `json:"name"`
    Age   int    `json:"age"`
    Email string `json:"email"`
}

// In-memory data storage
var (
    students = make(map[int]Student)
    mu       sync.Mutex
)

// Function to create a new student
func createStudent(w http.ResponseWriter, r *http.Request) {
    var student Student
    err := json.NewDecoder(r.Body).Decode(&student)
    if err != nil {
        http.Error(w, "Invalid input", http.StatusBadRequest)
        return
    }
    mu.Lock()
    student.ID = rand.Intn(1000) // Generate a random ID
    students[student.ID] = student
    mu.Unlock()
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(student)
}

// Function to get all students
func getAllStudents(w http.ResponseWriter, r *http.Request) {
    mu.Lock()
    defer mu.Unlock()
    var allStudents []Student
    for _, student := range students {
        allStudents = append(allStudents, student)
    }
    json.NewEncoder(w).Encode(allStudents)
}

// Function to get a student by ID
func getStudentByID(w http.ResponseWriter, r *http.Request) {
    id, err := strconv.Atoi(r.URL.Path[len("/students/"):])
    if err != nil {
        http.Error(w, "Invalid ID", http.StatusBadRequest)
        return
    }
    mu.Lock()
    student, exists := students[id]
    mu.Unlock()
    if !exists {
        http.Error(w, "Student not found", http.StatusNotFound)
        return
    }
    json.NewEncoder(w).Encode(student)
}

// Function to update a student by ID
func updateStudentByID(w http.ResponseWriter, r *http.Request) {
    id, err := strconv.Atoi(r.URL.Path[len("/students/"):])
    if err != nil {
        http.Error(w, "Invalid ID", http.StatusBadRequest)
        return
    }
    var updatedStudent Student
    err = json.NewDecoder(r.Body).Decode(&updatedStudent)
    if err != nil {
        http.Error(w, "Invalid input", http.StatusBadRequest)
        return
    }
    mu.Lock()
    _, exists := students[id]
    if exists {
        updatedStudent.ID = id
        students[id] = updatedStudent
    }
    mu.Unlock()
    if !exists {
        http.Error(w, "Student not found", http.StatusNotFound)
        return
    }
    json.NewEncoder(w).Encode(updatedStudent)
}

// Function to delete a student by ID
func deleteStudentByID(w http.ResponseWriter, r *http.Request) {
    id, err := strconv.Atoi(r.URL.Path[len("/students/"):])
    if err != nil {
        http.Error(w, "Invalid ID", http.StatusBadRequest)
        return
    }
    mu.Lock()
    delete(students, id)
    mu.Unlock()
    w.WriteHeader(http.StatusNoContent)
}

// Function to call Ollama API to generate a student summary
func callOllamaAPI(student Student) (string, error) {
    // Define the payload for Ollama's Llama3 model
    requestBody, err := json.Marshal(map[string]interface{}{
        "model": "llama3",
        "prompt": fmt.Sprintf("Generate a brief summary for a student named %s, aged %d, with email %s.",
            student.Name, student.Age, student.Email),
    })
    if err != nil {
        return "", err
    }

    // Make an HTTP POST request to Ollama's API endpoint
    resp, err := http.Post("http://localhost:11434/generate", "application/json", bytes.NewBuffer(requestBody))
    if err != nil {
        return "", err
    }
    defer resp.Body.Close()

    // Check if the response was successful
    if resp.StatusCode != http.StatusOK {
        return "", errors.New("failed to generate summary from Ollama API")
    }

    // Parse the response body
    body, err := ioutil.ReadAll(resp.Body)
    if err != nil {
        return "", err
    }

    // Extract summary from the response
    var response map[string]string
    if err := json.Unmarshal(body, &response); err != nil {
        return "", err
    }
    return response["summary"], nil
}

// Function to get a summary of a student by ID
func getStudentSummary(w http.ResponseWriter, r *http.Request) {
    id, err := strconv.Atoi(r.URL.Path[len("/students/"):])
    if err != nil {
        http.Error(w, "Invalid ID", http.StatusBadRequest)
        return
    }

    mu.Lock()
    student, exists := students[id]
    mu.Unlock()
    if !exists {
        http.Error(w, "Student not found", http.StatusNotFound)
        return
    }

    // Generate summary using Ollama
    summary, err := callOllamaAPI(student)
    if err != nil {
        http.Error(w, "Error generating summary", http.StatusInternalServerError)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(map[string]string{"summary": summary})
}

// Main function to route endpoints
func main() {
    http.HandleFunc("/students", func(w http.ResponseWriter, r *http.Request) {
        switch r.Method {
        case "POST":
            createStudent(w, r)
        case "GET":
            getAllStudents(w, r)
        default:
            http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
        }
    })

    http.HandleFunc("/students/", func(w http.ResponseWriter, r *http.Request) {
        if len(r.URL.Path) > len("/students/") {
            idPath := r.URL.Path[len("/students/"):]
            if len(idPath) > 8 && idPath[len(idPath)-8:] == "/summary" {
                getStudentSummary(w, r)
                return
            }
            switch r.Method {
            case "GET":
                getStudentByID(w, r)
            case "PUT":
                updateStudentByID(w, r)
            case "DELETE":
                deleteStudentByID(w, r)
            default:
                http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
            }
        } else {
            http.Error(w, "Invalid endpoint", http.StatusBadRequest)
        }
    })

    log.Println("Server starting on port 8080...")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
