# library-management-system
#include <iostream>
#include <string>
#include <fstream>
#include <vector>
#include <iomanip>
#include <algorithm>
#include <ctime>
#include <cctype>
#include <map>
#include <filesystem>
using namespace std;
namespace fs = filesystem;

// ===============================
// 🔹 Utility Functions
// ===============================
bool isAlphabetic(const string &str) {
    if (str.empty()) return false;
    for (char c : str)
        if (!isalpha(c) && !isspace(c))
            return false;
    return true;
}

bool isAlphaNumeric(const string &str) {
    if (str.empty()) return false;
    for (char c : str)
        if (!isalnum(c))
            return false;
    return true;
}

string toLower(string s) {
    transform(s.begin(), s.end(), s.begin(), ::tolower);
    return s;
}

string currentTime() {
    time_t now = time(0);
    string t = ctime(&now);
    t.pop_back();
    return t;
}

void ensureFilesExist() {
    vector<string> files = {"books.txt", "members.txt", "history.txt"};
    for (auto &f : files) {
        if (!fs::exists(f)) {
            ofstream create(f);
            create.close();
        }
    }
}

// ===============================
// 🔹 Abstract Classes
// ===============================
class Displayable {
public:
    virtual void display() const = 0;
    virtual ~Displayable() {}
};

class FileHandler {
public:
    virtual void save() = 0;
    virtual void load() = 0;
    virtual ~FileHandler() {}
};

// ===============================
// 🔹 Person Class
// ===============================
class Person {
protected:
    string name;
public:
    Person() {}
    Person(string n) : name(n) {}
    string getName() const { return name; }
    void setName(string n) { name = n; }
};

// ===============================
// 🔹 Member Class
// ===============================
class Member : public Person, public Displayable {
private:
    string id;
    int borrowedCount;
public:
    Member() : borrowedCount(0) {}
    Member(string n, string i) : Person(n), id(i), borrowedCount(0) {}

    string getId() const { return id; }
    int getBorrowedCount() const { return borrowedCount; }
    void increaseBorrow() { borrowedCount++; }
    void decreaseBorrow() { if (borrowedCount > 0) borrowedCount--; }

    void display() const override {
        cout << "ID: " << id << " | Name: " << name
             << " | Borrowed Books: " << borrowedCount << endl;
    }

    string toFileString() const {
        return id + "|" + name + "|" + to_string(borrowedCount) + "\n";
    }

    void fromFileString(const string &line) {
        size_t p1 = line.find('|');
        size_t p2 = line.find('|', p1 + 1);
        id = line.substr(0, p1);
        name = line.substr(p1 + 1, p2 - p1 - 1);
        borrowedCount = stoi(line.substr(p2 + 1));
    }
};
// ===============================
// 🔹 Book Class
// ===============================
class Book : public Displayable {
private:
    string title, author, isbn;
    int totalCopies, copiesAvailable, borrowCount;
public:
    Book() : totalCopies(1), copiesAvailable(1), borrowCount(0) {}
    Book(string t, string a, string i, int copies = 1)
        : title(t), author(a), isbn(i), totalCopies(copies),
          copiesAvailable(copies), borrowCount(0) {}

    string getTitle() const { return title; }
    string getIsbn() const { return isbn; }
    int getAvailableCopies() const { return copiesAvailable; }
    bool isAvailable() const { return copiesAvailable > 0; }

    void borrowBook() {
        if (copiesAvailable > 0) {
            copiesAvailable--;
            borrowCount++;
        }
    }

    void returnBook() {
        if (copiesAvailable < totalCopies) copiesAvailable++;
    }

    void display() const override {
        cout << left << setw(25) << title
             << " | Author: " << setw(15) << author
             << " | ISBN: " << setw(10) << isbn
             << " | Copies: " << copiesAvailable << "/" << totalCopies << endl;
    }

    string toFileString() const {
        return title + "|" + author + "|" + isbn + "|" +
               to_string(totalCopies) + "|" + to_string(copiesAvailable) +
               "|" + to_string(borrowCount) + "\n";
    }

    void fromFileString(const string &line) {
        size_t p1 = line.find('|');
        size_t p2 = line.find('|', p1 + 1);
        size_t p3 = line.find('|', p2 + 1);
        size_t p4 = line.find('|', p3 + 1);
        size_t p5 = line.find('|', p4 + 1);
        title = line.substr(0, p1);
        author = line.substr(p1 + 1, p2 - p1 - 1);
        isbn = line.substr(p2 + 1, p3 - p2 - 1);
        totalCopies = stoi(line.substr(p3 + 1, p4 - p3 - 1));
        copiesAvailable = stoi(line.substr(p4 + 1, p5 - p4 - 1));
        borrowCount = stoi(line.substr(p5 + 1));
    }
};

// ===============================
// 🔹 Library Class
// ===============================
class Library : public FileHandler {
private:
    vector<Book> books;
    vector<Member> members;
    map<string, vector<string>> memberBooks;

    void logActivity(const string &action, const string &details) {
        ofstream log("history.txt", ios::app);
        log << "[" << currentTime() << "] " << action << ": " << details << endl;
        log.close();
    }

public:
    Library() { ensureFilesExist(); load(); }
    ~Library() { save(); }

    void save() override {
        ofstream bf("books.txt");
        for (auto &b : books) bf << b.toFileString();
        bf.close();

        ofstream mf("members.txt");
        for (auto &m : members) mf << m.toFileString();
        mf.close();
    }

    void load() override {
        books.clear();
        members.clear();
        string line;

        ifstream bf("books.txt");
        while (getline(bf, line)) if (!line.empty()) { Book b; b.fromFileString(line); books.push_back(b); }
        bf.close();

        ifstream mf("members.txt");
        while (getline(mf, line)) if (!line.empty()) { Member m; m.fromFileString(line); members.push_back(m); }
        mf.close();
    }
