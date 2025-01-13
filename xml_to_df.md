import xml.etree.ElementTree as ET
import pandas as pd
import psycopg2
from sqlalchemy import create_engine

# Parse the XML file
file_path = 'bce_extract.xml'  # Replace with your file path
ns = {'ns0': 'http://economie.fgov.be/kbo/extract/v1/extracts'}
tree = ET.parse(file_path)
root = tree.getroot()

# Helper function to safely extract text from an element
def get_text(element):
    return element.text if element is not None else None

# Extract header details
def parse_header():
    header = root.find('ns0:Header', ns)
    if header is not None:
        header_data = {child.tag.split('}')[-1]: get_text(child) for child in header}
        return pd.DataFrame([header_data])
    return pd.DataFrame()

# Extract cancelled business units
def parse_cancelled_units():
    cancelled_units = root.find('ns0:CancelledBusinessUnits', ns)
    if cancelled_units is not None:
        units = [get_text(child) for child in cancelled_units.findall('ns0:CancelledBusinessUnitNumber', ns)]
        return pd.DataFrame(units, columns=['CancelledBusinessUnitNumber'])
    return pd.DataFrame()

# Extract enterprise details
def parse_enterprises():
    enterprises = root.findall('ns0:Enterprises/ns0:Enterprise', ns)
    data = []
    for enterprise in enterprises:
        enterprise_data = {
            'Nbr': get_text(enterprise.find('ns0:Nbr', ns)),
            'RegistrationDate': get_text(enterprise.find('ns0:RegistrationDate', ns)),
            'Type': get_text(enterprise.find('ns0:Type', ns)),
            'Status': get_text(enterprise.find('ns0:Status', ns)),
            'Capital': get_text(enterprise.find('ns0:Capital', ns)),
            'Currency': get_text(enterprise.find('ns0:Currency', ns)),
        }
        data.append(enterprise_data)
    return pd.DataFrame(data)

# Extract business unit details
def parse_business_units():
    business_units = root.findall('ns0:BusinessUnits/ns0:BusinessUnit', ns)
    data = []
    for unit in business_units:
        unit_data = {
            'Number': get_text(unit.find('ns0:Number', ns)),
            'RegistrationDate': get_text(unit.find('ns0:RegistrationDate', ns)),
            'Status': get_text(unit.find('ns0:Status', ns)),
        }
        data.append(unit_data)
    return pd.DataFrame(data)

# Extract footer details
def parse_footer():
    footer = root.find('ns0:Footer', ns)
    if footer is not None:
        footer_data = {child.tag.split('}')[-1]: get_text(child) for child in footer}
        return pd.DataFrame([footer_data])
    return pd.DataFrame()

# Main processing
header_df = parse_header()
cancelled_units_df = parse_cancelled_units()
enterprises_df = parse_enterprises()
business_units_df = parse_business_units()
footer_df = parse_footer()

# Save or display results
print("Header Data:")
print(header_df.head())
print("\nCancelled Business Units:")
print(cancelled_units_df.head())
print("\nEnterprises:")
print(enterprises_df.head())
print("\nBusiness Units:")
print(business_units_df.head())
print("\nFooter Data:")
print(footer_df.head())

# Save data to PostgreSQL database
pg_connection_string = 'postgresql+psycopg2://username:password@host:port/database'  # Update with your PostgreSQL details
engine = create_engine(pg_connection_string)

header_df.to_sql('header', engine, if_exists='replace', index=False)
cancelled_units_df.to_sql('cancelled_business_units', engine, if_exists='replace', index=False)
enterprises_df.to_sql('enterprises', engine, if_exists='replace', index=False)
business_units_df.to_sql('business_units', engine, if_exists='replace', index=False)
footer_df.to_sql('footer', engine, if_exists='replace', index=False)

print("Data has been ingested into the PostgreSQL database.")
