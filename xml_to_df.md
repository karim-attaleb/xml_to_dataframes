### Parsing XML and Inserting Data into PostgreSQL

This document provides a Python-based solution for parsing an XML file and saving the parsed data into a PostgreSQL database. Below is the complete code and an explanation of each part.

#### Python Code

```python
import xml.etree.ElementTree as ET
import pandas as pd
import psycopg2
from sqlalchemy import create_engine
import logging
import uuid

# Configure logging
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')

# Parse the XML file
file_path = 'bce_extract.xml'  # Replace with your file path
ns = {'ns0': 'http://economie.fgov.be/kbo/extract/v1/extracts'}
logging.info("Parsing XML file: %s", file_path)
tree = ET.parse(file_path)
root = tree.getroot()

# Helper function to safely extract text from an element
def get_text(element):
    """Safely extract text from an XML element.

    Args:
        element (ET.Element): XML element.

    Returns:
        str or None: Text content of the element or None if the element is missing.
    """
    return element.text if element is not None else None

# Recursive function to parse all nested XML elements into normalized structures
def parse_element_to_rows(element, parent_id=None, parent_name=None):
    """Parse an XML element and its children into rows for normalized tables.

    Args:
        element (ET.Element): XML element to parse.
        parent_id (str): UUID of the parent row.
        parent_name (str): Name of the parent element.

    Yields:
        tuple: (table_name, row_data, child_element) representing the table, row, and nested children.
    """
    row_id = str(uuid.uuid4())  # Generate a unique ID for the current row
    row = {"id": row_id, "parent_id": parent_id, "parent_name": parent_name}
    table_name = element.tag.split('}')[-1]

    for child in element:
        tag = child.tag.split('}')[-1]
        if len(child):
            # If the element has children, recurse
            yield from parse_element_to_rows(child, parent_id=row_id, parent_name=table_name)
        else:
            row[tag] = get_text(child)

    yield table_name, row, element

# Normalize XML to multiple tables
def normalize_xml_to_tables(root):
    """Normalize XML into multiple related tables.

    Args:
        root (ET.Element): Root of the XML document.

    Returns:
        dict: Dictionary of table names to rows.
    """
    tables = {}
    for table_name, row, _ in parse_element_to_rows(root):
        if table_name not in tables:
            tables[table_name] = []
        tables[table_name].append(row)
    return tables

# Convert normalized tables to DataFrames
def tables_to_dataframes(tables):
    """Convert normalized tables into Pandas DataFrames and adjust column types.

    Args:
        tables (dict): Dictionary of table names to rows.

    Returns:
        dict: Dictionary of table names to DataFrames.
    """
    dataframes = {}
    for table, rows in tables.items():
        df = pd.DataFrame(rows)
        # Adjust column types
        for column in df.columns:
            if column.endswith("_id") or column == "id":
                df[column] = df[column].astype(str)  # Ensure IDs are stored as strings
            elif column.lower().endswith("date"):
                df[column] = pd.to_datetime(df[column], errors='coerce')
            elif column.lower() in ["amount", "capital"]:
                df[column] = pd.to_numeric(df[column], errors='coerce')
        dataframes[table] = df
    return dataframes

# Main processing
logging.info("Starting XML parsing and normalization")
normalized_tables = normalize_xml_to_tables(root)
dataframes = tables_to_dataframes(normalized_tables)
logging.info("Normalization complete")

# Save or display results
logging.info("Displaying parsed data")
for table_name, df in dataframes.items():
    print(f"\nTable: {table_name}")
    print(df.head())

# Save data to PostgreSQL database
logging.info("Saving data to PostgreSQL database")
pg_connection_string = 'postgresql+psycopg2://username:password@host:port/database'  # Update with your PostgreSQL details
engine = create_engine(pg_connection_string)

for table_name, df in dataframes.items():
    df.to_sql(table_name, engine, if_exists='replace', index=False)
    logging.info(f"Data for table '{table_name}' saved to PostgreSQL")

logging.info("Data successfully ingested into the PostgreSQL database")
```

### Explanation

1. **Namespaces and File Parsing:**
   - The `ns` dictionary defines the namespace for parsing.
   - The XML file is loaded and parsed using `xml.etree.ElementTree`.

2. **Recursive Parsing (`parse_element_to_rows`):**
   - Handles nested XML structures by recursively generating rows for normalized tables.
   - Uses `uuid.uuid4()` to create persistent unique identifiers for each row.

3. **Normalization (`normalize_xml_to_tables`):**
   - Converts XML data into multiple related tables.

4. **DataFrames and Column Types:**
   - Converts tables into DataFrames.
   - Adjusts column types: string for IDs, datetime for date fields, and numeric for amounts.

5. **Database Insertion:**
   - DataFrames are saved to PostgreSQL tables using SQLAlchemy, with table names derived from XML tags.

6. **Output:**
   - The script prints the first few rows of each DataFrame and confirms data ingestion into PostgreSQL.

### Prerequisites

1. **Install Required Libraries:**
   ```bash
   pip install pandas sqlalchemy psycopg2
   ```

2. **Update Connection String:**
   Replace `username`, `password`, `host`, `port`, and `database` in the connection string with your PostgreSQL credentials.

3. **Run the Script:**
   Execute the script to parse the XML file and insert the data into the PostgreSQL database.

### Notes
- Ensure the XML file path is correct.
- This approach normalizes nested XML structures into relational tables.
- Each table maintains references to parent rows for relational integrity.
- Columns are automatically typed for better database compatibility.
```

