# fd

// See https://aka.ms/new-console-template for more information
using System;
using System.Runtime.CompilerServices;
using Microsoft.Data.Sqlite;

class Program
{
    static string connectionString = "Data Source=salse.db";
    static void Main(string[] args)
    {
        //DB Path
        //string dbFile = "salse.db";
        //string connectionString = $"Data Source={dbFile}";

        //Initialize DB and tables
        InitializeDatabase(connectionString);

        Console.WriteLine("Welcome to SMS!");
        //Console.WriteLine("Ready to using DB!");
        //}

        //MainMenu(Loop)
        while (true)
        {
            Console.WriteLine("\n=== MenuCommand ===");
            Console.WriteLine("1. Register sales");
            Console.WriteLine("2. Show all sales");
            Console.WriteLine("3. Update sales");
            Console.WriteLine("4. Delete sales");
            Console.WriteLine("5. Exit");
            Console.WriteLine("Please select: ");

            string choice = Console.ReadLine();

            switch (choice)
            {
                case "1":
                    AddSales();
                    break;
                case "2":
                    ShowAllSales();
                    break;
                case "3":
                    UpdateSales();
                    break;
                case "4":
                    DeleteSales();
                    break;
                case "5":
                    Console.WriteLine("Exit");
                    return;
                default:
                    Console.WriteLine("Please select 1 - 5");
                    break;
            }
        }
    }

    //Function to create DB and tables
    static void InitializeDatabase(string connString)
    {
        //Create Tables
        using (var connection = new SqliteConnection(connString))
        {
            connection.Open();

            string createTableQuery = @"
            CREATE TABLE IF NOT EXISTS Sales (
            Id INTEGER PRIMARY KEY AUTOINCREMENT,
            ProductCode TEXT NOT NULL,
            Date TEXT NOT NULL,
            ProductName TEXT NOT NULL,
            SalesAmount REAL NOT NULL,
            Cost REAL NOT NULL,
            GrossProfit REAL NOT NULL,
            IsDeleted INTEGER DEFAULT 0
            )";

            using (var command = new SqliteCommand(createTableQuery, connection))
            {
                command.ExecuteNonQuery();
                Console.WriteLine("Prepared to Sales Table.");
            }
        }
    }

    //Function to register Sales data
    static void AddSales()
    {
        Console.WriteLine("\n=== Register Sales ===");

        //Input Product Code
        Console.Write("Product Code: ");
        string productCode = Console.ReadLine();

        if (string.IsNullOrWhiteSpace(productCode))
        {
            Console.WriteLine("Product Code is required.");
            return;
        }

        //Check input date type
        DateTime date;
        while (true)
        {
            Console.Write("Date (YYYY-MM-DD): ");
            string dateInput = Console.ReadLine();

            if (DateTime.TryParseExact(dateInput, "yyyy-MM-dd", null,
                System.Globalization.DateTimeStyles.None, out date))
            {
                Console.WriteLine($"Parsed date: {date:yyyy-MM-dd}");
                break; //If the input is valid, exit the loop 
            }
            else
            {
                Console.WriteLine("Invalid format. Please use YYYY-MM-DD (e.g., 2026-02-09");
            }
        }


        Console.Write("Product Name: ");
        string productName = Console.ReadLine();

        Console.Write("Amount: ");
        if (!double.TryParse(Console.ReadLine(), out double salesAmount))
        {
            Console.Write("Please input number.");
            return;
        }

        Console.Write("Cost: ");
        if (!double.TryParse(Console.ReadLine(), out double cost))
        {
            Console.Write("Pleas input number.");
            return;
        }

        //Calcurate gross profit
        double grossProfit = salesAmount - cost;

        //Insert into DB
        using (var connection = new SqliteConnection(connectionString))
        {
            connection.Open();

            string insertQuery = @"
            INSERT INTO Sales (ProductCode, DATE, ProductName, SalesAmount, Cost, GrossProfit, IsDeleted)
            VALUES (@productCode, @date, @productName, @salesAmount, @cost, @grossProfit, 0)";

            using (var command = new SqliteCommand(insertQuery, connection))
            {

                command.Parameters.AddWithValue("@productCode", productCode);
                command.Parameters.AddWithValue("@date", date.ToString("yyyy-MM-dd"));
                command.Parameters.AddWithValue("@productName", productName);
                command.Parameters.AddWithValue("@salesAmount", salesAmount);
                command.Parameters.AddWithValue("@cost", cost);
                command.Parameters.AddWithValue("@grossProfit", grossProfit);

                command.ExecuteNonQuery();
                Console.WriteLine($"\nDone! Gross Profit: {grossProfit:C}");
            }
        }
    }

    //Function to show all sales data
    static void ShowAllSales()
    {
        Console.WriteLine("\n=== List of sales data");

        using (var connection = new SqliteConnection(connectionString))
        {
            connection.Open();

            //Show only IsDeleted = 0
            string selectQuery = "SELECT * FROM Sales WHERE IsDeleted = 0 ORDER BY Date DESC";

            using (var command = new SqliteCommand(selectQuery, connection))
            using (var reader = command.ExecuteReader())
            {
                if (!reader.HasRows)
                {
                    Console.WriteLine("No data");
                    return;
                }

                Console.WriteLine($"{"ID",-5} {"date",-12} {"productName",-20} {"SalesAmount",-12} {"cost",-12} {"GrossProfit",-12:C}");
                Console.WriteLine(new string('-', 80));

                while (reader.Read())
                {
                    //To show type: yyyy-mm-dd by reading DateTime
                    DateTime date = DateTime.Parse(reader["Date"].ToString());

                    Console.WriteLine($"{reader["Id"],-5} {date.ToString("yyyy-MM-dd"),-12} {reader["ProductName"],-20} " +
                        $"{reader["SalesAmount"],-12:N0} {reader["Cost"],-12:N0} {reader["GrossProfit"],-12:N0}");
                }

            }
        }
    }
    static void DeleteSales()
    {

        Console.WriteLine("\n=== Delete sales ===");

        //First of all, show all sales data
        ShowAllSalesForSelection();

        Console.Write("\nEnter ID to delete ( or 0 to cancel): ");
        if (!int.TryParse(Console.ReadLine(), out int id))
        {
            Console.WriteLine("Inavalid ID. ");
            return;
        }

        if (id == 0)
        {
            Console.WriteLine("Cancelled.");
            return;
        }

        //Confirmation before delete
        if (!ConfirmDelete(id))
        {
            Console.WriteLine("Cancelled.");
            return;
        }

        //Logical deletion (Alter IsDeleted = 1)
        using (var connection = new SqliteConnection(connectionString))
        {
            connection.Open();

            string updateQuery = "UPDATE Sales SET IsDeleted = 1 WHERE ID = @id";

            using (var command = new SqliteCommand(updateQuery, connection))
            {
                command.Parameters.AddWithValue("@id", id);

                int rowsAffected = command.ExecuteNonQuery();

                if (rowsAffected > 0)
                {
                    Console.WriteLine("$\nID {id} was successfully deleted!");
                }
                else
                {
                    Console.WriteLine($"\nID {id} not found. ");

                }
            }
        }

    }
    //Function to update sales data
    static void UpdateSales()
    {
        Console.WriteLine("\n=== Update Sales === ");

        //First of all, Show all sales data
        ShowAllSalesForSelection();

        Console.Write("\nEnter ID to update (or 0 to cancel): ");
        if (!int.TryParse(Console.ReadLine(), out int id))
        {
            Console.WriteLine("Invalid ID.");
            return;
        }

        //To get and show latest data
        var currentData = GetSalesData(id);
        if (currentData == null)
        {
            Console.WriteLine($"ID {id} not found");
            return;
        }

        //Show latest Values
        Console.WriteLine("\nCurrent data: ");
        Console.WriteLine($"Product Code: {currentData.ProductCode}");
        Console.WriteLine($"Date: {currentData.Date}");
        Console.WriteLine($"Product Name: {currentData.ProductName}");
        Console.WriteLine($"Product Code: {currentData.ProductCode}");
        Console.WriteLine($"Sales Amount: {currentData.SalesAmount:N0}");
        Console.WriteLine($"Cost: {currentData.Cost:N0}");
        Console.WriteLine($"Gross Profit: {currentData.GrossProfit:N0}");

        Console.WriteLine("\nEnter new values (press Enter to keep current value): ");

        //New Product Code
        Console.Write($"Product Code[{currentData.ProductCode}]: ");
        string newProductCode = Console.ReadLine();
        if (string.IsNullOrWhiteSpace(newProductCode))
        {
            newProductCode = currentData.ProductCode;
        }

        //New Date
        DateTime newDate;

        while (true)
        {
            Console.Write($"Date (YYYY-MM-DD) [{currentData.Date}]: ");
            string dateInput = Console.ReadLine();

            if (string.IsNullOrWhiteSpace(dateInput))
            {
                newDate = DateTime.Parse(currentData.Date);
                break;
            }

            if (DateTime.TryParseExact(dateInput, "yyyy-MM-dd", null,
                System.Globalization.DateTimeStyles.None, out newDate))
            {
                break;
            }
            else
            {
                Console.WriteLine("Invalid format. Please use YYYY-MM-DD");
            }
        }

        //New product name
        Console.Write($"Prduct Name [{currentData.ProductName}]: ");
        string newProductName = Console.ReadLine();
        if (string.IsNullOrWhiteSpace(newProductName))
        {
            newProductName = currentData.ProductName;
        }

        //NewSalesAmount
        double newSalesAmount;
        while (true)
        {
            Console.Write($"Sales Amount [{currentData.SalesAmount:N0}]: ");
            string amountInput = Console.ReadLine();

            if (string.IsNullOrWhiteSpace(amountInput))
            {
                newSalesAmount = currentData.SalesAmount;
                break;
            }

            if (double.TryParse(amountInput, out newSalesAmount) && newSalesAmount >= 0)
            {
                break;
            }
            else
            {
                Console.WriteLine("Please enter a valid positive number.");
            }
        }

        //New Cost
        double newCost;
        while (true)
        {
            Console.Write($"Cost[{currentData.Cost:N0}]: ");
            string costInput = Console.ReadLine();

            if (string.IsNullOrWhiteSpace(costInput))
            {
                newCost = currentData.Cost;
                break;
            }

            if (double.TryParse(costInput, out newCost) && newCost >= 0)
            {
                break;
            }
            else
            {
                Console.WriteLine("Please enter a valid positive number");
            }

        }
        //Reculcurate GrossProfit
        double newGrossProfit = newSalesAmount - newCost;

        //Confirm
        Console.WriteLine("\nNew data");
        Console.WriteLine($"Product Code: {newProductCode}");
        Console.WriteLine($"Date: {newDate:yyyy-MM-dd}");
        Console.WriteLine($"Product Name: {newProductName}");
        Console.WriteLine($"Sales Amount: {newSalesAmount:N0}");
        Console.WriteLine($"Cost: {newCost:N0}");
        Console.WriteLine($"Gross Profit: {newGrossProfit:N0}");

        Console.Write("\nUpdate this data? (yes/no): ");
        if (Console.ReadLine()?.ToLower() != "yes")
        {
            Console.WriteLine("Cancelled.");
            return;
        }

        //Update DB
        using (var connection = new SqliteConnection(connectionString))
        {
            connection.Open();

            string updateQuery = @"
                UPDATE Sales
                SET ProductCode = @productCode,
                    Date = @date,
                    ProductName = @productName,
                    SalesAmount = @salesAmount,
                    Cost = @cost,
                    GrossProfit = @grossProfit
                WHERE Id = @id";

            using (var command = new SqliteCommand(updateQuery, connection))
            {

                command.Parameters.AddWithValue("@productCode", newProductCode);
                command.Parameters.AddWithValue("@date", newDate.ToString("yyyy-MM-dd"));
                command.Parameters.AddWithValue("@productName", newProductName);
                command.Parameters.AddWithValue("@salesAmount", newSalesAmount);
                command.Parameters.AddWithValue("@cost", newCost);
                command.Parameters.AddWithValue("@grossProfit", newGrossProfit);
                command.Parameters.AddWithValue("@id", id);

                int rowsAffected = command.ExecuteNonQuery();

                if (rowsAffected > 0)
                {
                    Console.WriteLine($"\nID {id} was to successfully updated!");
                }
                else
                {
                    Console.WriteLine($"\nFailed to update ID {id}.");
                }

            }
        }
    }


    // Helper: Show List of deleted data
    static void ShowAllSalesForSelection()
    {
        using (var connection = new SqliteConnection(connectionString))
        {
            connection.Open();

            //Show only IsDeleted = 0
            string selectQuery = "SELECT * FROM Sales WHERE IsDeleted = 0 ORDER BY id";

            using (var command = new SqliteCommand(selectQuery, connection))
            using (var reader = command.ExecuteReader())
            {
                if (!reader.HasRows)
                {
                    Console.WriteLine("No data to delete.");
                    return;
                }

                Console.WriteLine($"{"ID",-5} {"ProductCode",-10}{"Date",-12} {"ProductName",-20} {"GrossProfit",-12}");
                Console.WriteLine(new string('-', 50));

                while (reader.Read())
                {
                    DateTime date = DateTime.Parse(reader["Date"].ToString());
                    Console.WriteLine($"{reader["Id"],-5} {reader["ProductCode"],-10} {date.ToString("yyyy-MM-dd"),-12} {reader["ProductName"],-20} {reader["GrossProfit"],-12:N0}");
                }
            }
        }
    }

    //Helper: Get sales data by ID
    static SalesData GetSalesData(int id)
    {
        using (var connection = new SqliteConnection(connectionString))
        {
            connection.Open();

            string selectQuery = "SELECT * FROM Sales WHERE Id = @id AND IsDeleted = 0";

            using (var command = new SqliteCommand(selectQuery, connection))
            {
                command.Parameters.AddWithValue("@id", id);

                using (var reader = command.ExecuteReader())
                {
                    if (reader.Read())
                    {
                        return new SalesData
                        {
                            Id = Convert.ToInt32(reader["Id"]),
                            ProductCode = reader["ProductCode"].ToString(),
                            Date = reader["Date"].ToString(),
                            ProductName = reader["ProductName"].ToString(),
                            SalesAmount = Convert.ToDouble(reader["SalesAmount"]),
                            Cost = Convert.ToDouble(reader["Cost"]),
                            GrossProfit = Convert.ToDouble(reader["GrossProfit"])
                        };
                    }
                }
            }
        }

        return null;
    }

    //Helper class to hold sales data
    class SalesData
    {
        public int Id { get; set; }
        public string ProductCode { get; set; }
        public string Date { get; set; }
        public string ProductName { get; set; }
        public double SalesAmount { get; set; }
        public double Cost { get; set; }
        public double GrossProfit { get; set; }
    }

    //Helper: Confirmation before delete
    static bool ConfirmDelete(int id) 
    {
        //Show to delete data
        using (var connection = new SqliteConnection(connectionString))
        {
            connection.Open();

            //Only data has IsDeleted = 0
            string selectQuery = "SELECT * FROM Sales WHERE Id = @id AND IsDeleted = 0";

            using (var command = new SqliteCommand(selectQuery, connection))
            {

                command.Parameters.AddWithValue("@id", id);

                using (var reader = command.ExecuteReader())
                {
                    if (!reader.Read())
                    {
                        return false;
                    }

                    DateTime date = DateTime.Parse(reader["Date"].ToString());
                    Console.WriteLine($"\nAre you sure you want to delete this?");
                    Console.WriteLine($"ID: {reader["Id"]}");
                    Console.WriteLine($"ID: {reader["ProductCode"]}");
                    Console.WriteLine($"Date: {date:yyyy-MM-dd}");
                    Console.WriteLine($"Product: {reader["ProductName"]}");
                    Console.WriteLine($"Sales: {reader["SalesAmount"]:N0}");
                    Console.WriteLine($"Gross Profit: {reader["GrossProfit"]:N0}");

                }
            }
        }

        Console.Write("\nType 'yes' to confirm: ");
        string confirmation = Console.ReadLine();

        return confirmation?.ToLower() == "yes";
        
    }

}
