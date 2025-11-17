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
    // ===============================
    // ADD BOOK WITHOUT EXCEPTION
    // ===============================
    void addBook() {
        cin.ignore();
        string title, author, isbn;
        int copies;

        cout << "Enter Book Title (alphabets/numbers allowed): "; getline(cin, title);
        if (title.empty()) { cout << "❌ Title cannot be empty!\n"; return; }

        cout << "Enter Author Name (alphabets only): "; getline(cin, author);
        if (!isAlphabetic(author)) { cout << "❌ Author must contain alphabets only!\n"; return; }

        cout << "Enter ISBN (alphanumeric allowed): "; getline(cin, isbn);
        if (!isAlphaNumeric(isbn)) { cout << "❌ ISBN must be alphanumeric!\n"; return; }

        for (auto &b : books) {
            if (toLower(b.getIsbn()) == toLower(isbn)) {
                cout << "❌ ISBN already exists!\n";
                return;
            }
        }

        cout << "Enter number of copies: ";
        while (!(cin >> copies) || copies <= 0) { 
            cout << "❌ Invalid number! Enter again: "; 
            cin.clear(); cin.ignore(10000,'\n'); 
        }

        books.push_back(Book(title, author, isbn, copies));
        cout << "✅ Book added successfully!\n";
        logActivity("Book Added", title);
        save();
    }

    void deleteBook() {
        if (books.empty()) { cout << "No books to delete.\n"; return; }
        displayBooks();
        int index;
        cout << "Enter Book Number to Delete: "; cin >> index;
        if (index < 1 || index > (int)books.size()) { cout << "❌ Invalid choice.\n"; return; }

        string deletedTitle = books[index-1].getTitle();
        books.erase(books.begin()+index-1);
        cout << "✅ Book \"" << deletedTitle << "\" deleted successfully!\n";
        logActivity("Book Deleted", deletedTitle);
        save();
    }

    void registerMember() {
        cin.ignore();
        string name, id;
        cout << "Enter Member Name (alphabets only): "; getline(cin, name);
        if (!isAlphabetic(name)) { cout << "❌ Invalid member name!\n"; return; }

        cout << "Enter Member ID (alphanumeric allowed): "; getline(cin, id);
        if (!isAlphaNumeric(id)) { cout << "❌ Member ID must be alphanumeric!\n"; return; }

        for (auto &m : members) {
            if (toLower(m.getId()) == toLower(id)) {
                cout << "❌ Member ID already exists!\n"; 
                return;
            }
        }

        members.push_back(Member(name, id));
        cout << "✅ Member registered successfully!\n";
        logActivity("Member Registered", name);
        save();
    }

    void borrowBook() {
        if (books.empty() || members.empty()) { cout << "No books or members available.\n"; return; }
        cin.ignore();
        string id; cout << "Enter Member ID: "; getline(cin, id);

        auto mem = find_if(members.begin(), members.end(), [&](Member &m){ return m.getId()==id; });
        if (mem==members.end()) { cout << "Member not found.\n"; return; }

        displayBooks();
        int index; cout << "Enter Book Number to Borrow: "; cin >> index;
        if (index<1 || index>(int)books.size()) { cout << "Invalid book selection.\n"; return; }

        Book &b = books[index-1];
        if (!b.isAvailable()) { cout << "Book not available.\n"; return; }

        b.borrowBook(); mem->increaseBorrow(); memberBooks[mem->getId()].push_back(b.getTitle());
        cout << "✅ " << mem->getName() << " borrowed \"" << b.getTitle() << "\".\n";
        logActivity("Book Borrowed", mem->getName() + " -> " + b.getTitle());
        save();
    }
    void returnBook() {
        cin.ignore(); string id; cout << "Enter Member ID: "; getline(cin, id);

        auto mem = find_if(members.begin(), members.end(), [&](Member &m){ return m.getId()==id; });
        if (mem==members.end()) { cout << "Member not found.\n"; return; }
        if (memberBooks[id].empty()) { cout << "⚠ This member has no borrowed books!\n"; return; }

        cout << "\nBooks borrowed by " << mem->getName() << ":\n";
        for (size_t i=0;i<memberBooks[id].size();i++) cout << i+1 << ". " << memberBooks[id][i] << endl;

        int index; cout << "Enter Book Number to Return: "; cin >> index;
        if (index<1 || index>(int)memberBooks[id].size()) { cout << "❌ Invalid choice.\n"; return; }

        string bookTitle = memberBooks[id][index-1];
        auto bookIt = find_if(books.begin(), books.end(), [&](Book &b){ return toLower(b.getTitle())==toLower(bookTitle); });

        if (bookIt!=books.end()) {
            bookIt->returnBook();
            mem->decreaseBorrow();
            memberBooks[id].erase(memberBooks[id].begin()+index-1);
            cout << "✅ " << mem->getName() << " returned \"" << bookIt->getTitle() << "\".\n";
            logActivity("Book Returned", mem->getName() + " -> " + bookIt->getTitle());
            save();
        }
    }

    void displayBooks() const {
        if (books.empty()) { cout << "No books available.\n"; return; }
        cout << "\n📚 BOOK LIST:\n";
        for (size_t i=0;i<books.size();i++) { cout << setw(2)<<i+1<<". "; books[i].display(); }
    }

    void displayMembers() const {
        if (members.empty()) { cout << "No members registered.\n"; return; }
        cout << "\n👤 MEMBER LIST:\n";
        for (auto &m : members) m.display();
    }

    void searchBook() {
        cin.ignore();
        string key; cout << "Enter Book Title or ISBN: "; getline(cin, key);
        bool found=false;
        for (auto &b : books) if (toLower(b.getTitle())==toLower(key) || b.getIsbn()==key) { cout<<"✅ Found: "; b.display(); found=true; }
        if (!found) cout << "❌ No match found.\n";
    }
};

// ===============================
// 🔹 MAIN FUNCTION
// ===============================
int main() {
    Library lib; int choice;

    while (true) {
        cout << "\n========== 📘 LIBRARY MANAGEMENT SYSTEM ==========\n";
        cout << "1. Add Book\n2. View Books\n3. Register Member\n4. View Members\n";
        cout << "5. Borrow Book\n6. Return Book\n7. Search Book\n8. Delete Book\n9. Exit\n";
        cout << "Enter your choice: ";

        if (!(cin >> choice)) { cout << "❌ Invalid input! Enter a number.\n"; cin.clear(); cin.ignore(10000,'\n'); continue; }

        switch(choice) {
            case 1: lib.addBook(); break;
            case 2: lib.displayBooks(); break;
            case 3: lib.registerMember(); break;
            case 4: lib.displayMembers(); break;
            case 5: lib.borrowBook(); break;
            case 6: lib.returnBook(); break;
            case 7: lib.searchBook(); break;
            case 8: lib.deleteBook(); break;
            case 9: cout << "💾 Data saved successfully. Goodbye!\n"; return 0;
            default: cout << "❌ Invalid choice.\n";
        }
    }
}
