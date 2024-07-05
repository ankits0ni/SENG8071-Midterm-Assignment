# Midterm Assignment

## Group Involvement

### Duties Assigned to Individual Members

**Ankit Soni (8972159):**
- Database Schema Design
- SQL Queries for requirements
- Implementation of CRUD operations

**Jay Sindhal (8963474):**
- Creation of TypeScript Interface
- Implementation of TypeScript functionalities
- Code Review and Testing

## Database Schema

### Authors Table

| Attribute | Type               |
|-----------|--------------------|
| author_id | SERIAL PRIMARY KEY |
| name      | VARCHAR(255)       |
| bio       | TEXT               |

### Publishers Table

| Attribute    | Type               |
|--------------|--------------------|
| publisher_id | SERIAL PRIMARY KEY |
| name         | VARCHAR(255)       |
| address      | TEXT               |

### Genres Table

| Attribute | Type               |
|-----------|--------------------|
| genre_id  | SERIAL PRIMARY KEY |
| name      | VARCHAR(255)       |

### Books Table

| Attribute      | Type               |
|----------------|--------------------|
| book_id        | SERIAL PRIMARY KEY |
| title          | VARCHAR(255)       |
| genre_id       | INT                |
| price          | DECIMAL(10,2)      |
| format         | VARCHAR(20)        |
| author_id      | INT                |
| publisher_id   | INT                |
| published_date | DATE               |
| stock_quantity | INT                |

### Customers Table

| Attribute   | Type               |
|-------------|--------------------|
| customer_id | SERIAL PRIMARY KEY |
| name        | VARCHAR(255)       |
| email       | VARCHAR(255)       |
| total_spent | DECIMAL(10,2)      |

### Reviews Table

| Attribute    | Type               |
|--------------|--------------------|
| review_id    | SERIAL PRIMARY KEY |
| book_id      | INT                |
| customer_id  | INT                |
| rating       | INT                |
| review_text  | TEXT               |
| review_date  | DATE               |

## SQL Queries

### Power Writers

```sql
SELECT author_id, name 
FROM Authors 
WHERE author_id IN (
  SELECT author_id 
  FROM Books 
  WHERE genre_id = $1 
    AND published_date >= NOW() - INTERVAL '$2 years'
  GROUP BY author_id 
  HAVING COUNT(book_id) > $3
);
```

###Loyal Customers
```sql
SELECT customer_id, name, email, total_spent 
FROM Customers 
WHERE total_spent > $1 
  AND NOW() - INTERVAL '1 year';
```

###Well Reviewed Books
```sql
SELECT book_id, title 
FROM Books 
WHERE book_id IN (
  SELECT book_id 
  FROM Reviews 
  GROUP BY book_id 
  HAVING AVG(rating) > (
    SELECT AVG(rating) 
    FROM Reviews
  )
);
```

###Most Popular Genre by Sales
```sql
SELECT genre_id, name 
FROM Genres 
WHERE genre_id = (
  SELECT genre_id 
  FROM Books 
  GROUP BY genre_id 
  ORDER BY SUM(stock_quantity) DESC 
  LIMIT 1
);
```

###10 Most Recent Posted Reviews by Customers
```sql
SELECT review_id, book_id, customer_id, rating, review_text, review_date 
FROM Reviews 
ORDER BY review_date DESC 
LIMIT 10;
```
###TypeScript Interface and Implementation
```sql
import { Pool, QueryResult } from 'pg';

// Database configuration
const pool = new Pool({
  user: 'postgres', // Replace with your PostgreSQL username
  host: 'localhost',
  database: 'bookstore', // Replace with your PostgreSQL database name
  password: 'password', // Replace with your PostgreSQL password
  port: 5432,
});

// Interfaces for database entities
interface Book {
  book_id: number;
  title: string;
  genre_id: number;
  price: number;
  format: string;
  author_id: number;
  publisher_id: number;
  published_date: Date;
  stock_quantity: number;
}

interface CreateBookDTO {
  title: string;
  genre_id: number;
  price: number;
  format: string;
  author_id: number;
  publisher_id: number;
  published_date: Date;
  stock_quantity: number;
}

// BookRepository class
class BookRepository {
  private pool: Pool;

  constructor(pool: Pool) {
    this.pool = pool;
  }

  async createBook(book: CreateBookDTO): Promise<Book> {
    const query = `
      INSERT INTO Books (title, genre_id, price, format, author_id, publisher_id, published_date, stock_quantity)
      VALUES ($1, $2, $3, $4, $5, $6, $7, $8)
      RETURNING *
    `;
    const values = [
      book.title,
      book.genre_id,
      book.price,
      book.format,
      book.author_id,
      book.publisher_id,
      book.published_date,
      book.stock_quantity,
    ];

    try {
      const result: QueryResult = await this.pool.query(query, values);
      return result.rows[0];
    } catch (error) {
      console.error('Error creating book:', error);
      throw error;
    }
  }

  async getBookById(bookId: number): Promise<Book | null> {
    const query = 'SELECT * FROM Books WHERE book_id = $1';
    const values = [bookId];

    try {
      const result: QueryResult = await this.pool.query(query, values);
      return result.rows.length ? result.rows[0] : null;
    } catch (error) {
      console.error(`Error fetching book with ID ${bookId}:`, error);
      throw error;
    }
  }

  async updateBook(bookId: number, updatedBook: Partial<CreateBookDTO>): Promise<Book | null> {
    const query = `
      UPDATE Books
      SET title = $1,
          genre_id = $2,
          price = $3,
          format = $4,
          author_id = $5,
          publisher_id = $6,
          published_date = $7,
          stock_quantity = $8
      WHERE book_id = $9
      RETURNING *
    `;
    const values = [
      updatedBook.title,
      updatedBook.genre_id,
      updatedBook.price,
      updatedBook.format,
      updatedBook.author_id,
      updatedBook.publisher_id,
      updatedBook.published_date,
      updatedBook.stock_quantity,
      bookId,
    ];

    try {
      const result: QueryResult = await this.pool.query(query, values);
      return result.rows.length ? result.rows[0] : null;
    } catch (error) {
      console.error(`Error updating book with ID ${bookId}:`, error);
      throw error;
    }
  }

  async deleteBook(bookId: number): Promise<boolean> {
    const query = 'DELETE FROM Books WHERE book_id = $1';
    const values = [bookId];

    try {
      const result: QueryResult = await this.pool.query(query, values);
      return result.rowCount !== null && result.rowCount > 0;
    } catch (error) {
      console.error(`Error deleting book with ID ${bookId}:`, error);
      throw error;
    }
  }

  async getAllBooks(): Promise<Book[]> {
    const query = 'SELECT * FROM Books';
    try {
      const result: QueryResult = await this.pool.query(query);
      return result.rows;
    } catch (error) {
      console.error('Error fetching books:', error);
      throw error;
    }
  }
}

async function initializeDatabase() {
  // Create tables if they don't exist
  const createTablesQuery = `
    CREATE TABLE IF NOT EXISTS Books (
      book_id SERIAL PRIMARY KEY,
      title VARCHAR(255),
      genre_id INT,
      price DECIMAL(10,2),
      format VARCHAR(20),
      author_id INT,
      publisher_id INT,
      published_date DATE,
      stock_quantity INT
    );
    -- Add other tables (Authors, Publishers, Genres, etc.) similarly
  `;
  
  try {
    await pool.query(createTablesQuery);
    console.log('Tables created successfully.');

    // Insert sample data if needed
    const bookRepo = new BookRepository(pool);
    const newBook: CreateBookDTO = {
      title: 'Fictional Book',
      genre_id: 1,
      price: 25.00,
      format: 'paperback',
      author_id: 1,
      publisher_id: 1,
      published_date: new Date('2024-01-01'),
      stock_quantity: 50,
    };
    const createdBook = await bookRepo.createBook(newBook);
    console.log('Created book:', createdBook);

    // Retrieve all books
    const allBooks = await bookRepo.getAllBooks();
    console.log('All books:', allBooks);

    // Example: Get book by ID
    const bookById = await bookRepo.getBookById(1);
    console.log('Book with ID 1:', bookById);

    // Example: Update book
    if (bookById) {
      const updatedBookData: Partial<CreateBookDTO> = {
        ...bookById,
        price: 30.00,
        format: 'hardcover',
        stock_quantity: 100,
      };
      const updatedBook = await bookRepo.updateBook(1, updatedBookData);
      console.log('Updated book:', updatedBook);
    }

    // Example: Delete book
    const deleteResult = await bookRepo.deleteBook(1);
    console.log('Book deleted successfully:', deleteResult);

  } catch (error) {
    console.error('Error initializing database:', error);
    throw error;
  }
}

async function main() {
  try {
    await initializeDatabase();
  } catch (error) {
    console.error('An error occurred:', error);
  } finally {
    await pool.end();
  }
}

main();
```
