import streamlit as stAdd commentMore actions
import mysql.connector
from mysql.connector import Error
import pandas as pd


# Function to create a database connection
def create_connection():
    connection = None
    
    try:
        connection = mysql.connector.connect(
            host='localhost',
            user='root',
            password='christy@123',
            database='phone'
        )
        if connection.is_connected():
            st.success("Connection to MySQL DB successful")
    except Error as e:
        st.error(f"The error '{e}' occurred")
    return connection


# Function to load contacts from the database
def load_contacts(connection):
    query = "SELECT Name, Mobile FROM contacts"
    cursor = connection.cursor()
    cursor.execute(query)
    result = cursor.fetchall()
    return pd.DataFrame(result, columns=['Name', 'Mobile'])


# Function to add a contact to the database
def add_contact(connection, name, mobile):
    cursor = connection.cursor()
    try:
        query = "INSERT INTO contacts (Name, Mobile) VALUES (%s, %s)"
        cursor.execute(query, (name, mobile))
        connection.commit()
        return True
    except mysql.connector.Error as err:
        if err.errno == mysql.connector.errorcode.ER_DUP_ENTRY:
            return False
        else:
            st.error(f"Error: {err}")
            return False


# Function to update contact name in the database
def update_contact_name(connection, old_name, new_name):
    cursor = connection.cursor()
    query = "UPDATE contacts SET Name = %s WHERE Name = %s"
    cursor.execute(query, (new_name, old_name))
    connection.commit()


# Function to update mobile number in the database
def update_contact_mobile(connection, name, new_mobile):
    cursor = connection.cursor()
    try:
        query = "SELECT * FROM contacts WHERE Mobile = %s AND Name != %s"
        cursor.execute(query, (new_mobile, name))
        if cursor.fetchone():
            st.warning('This mobile number already exists!')
            return
        query = "UPDATE contacts SET Mobile = %s WHERE Name = %s"
        cursor.execute(query, (new_mobile, name))
        connection.commit()
        st.success(f"Mobile number updated for {name}!")
    except mysql.connector.Error as err:
        st.error(f"Error: {err}")



# Function to delete a contact from the database
def delete_contact(connection, name):
    cursor = connection.cursor()
    query = "DELETE FROM contacts WHERE Name = %s"
    cursor.execute(query, (name,))
    connection.commit()


# Streamlit UI
st.title("Contact Book")

connection = create_connection()
contacts_df = load_contacts(connection)

choice = st.sidebar.selectbox("Choose an action",
                              ['Add Contact', 'Edit Name', 'Edit Mobile Number', 'Delete Contact', 'View All Contacts',
                               'Search Contacts'])

if choice == 'Add Contact':
    st.header("Add a new contact")
    name = st.text_input("Enter name:")
    mobile = st.text_input("Enter mobile number:")

    if st.button("Add Contact"):
        if add_contact(connection, name, mobile):
            st.success(f"Contact {name} added!")
            contacts_df = load_contacts(connection)  # Reload contacts
        else:
            st.warning('This mobile number already exists! ')

elif choice == 'Edit Name':
    st.header("Edit Contact Name")
    selected_name = st.selectbox("Select a contact to edit", contacts_df['Name'].unique())
    new_name = st.text_input("Enter new name:")

    if st.button("Update Name"):
        update_contact_name(connection, selected_name, new_name)
        st.success(f"Name updated to {new_name}!")
        contacts_df = load_contacts(connection)  # Reload contacts

elif choice == 'Edit Mobile Number':
    st.header("Edit Mobile Number")
    selected_name = st.selectbox("Select a contact", contacts_df['Name'].unique())
    new_mobile = st.text_input("Enter new mobile number:")

    if st.button("Update Mobile"):
        update_contact_mobile(connection, selected_name, new_mobile)
        contacts_df = load_contacts(connection)  # Reload contacts

elif choice == 'Delete Contact':
    st.header("Delete a Contact")
    selected_name = st.selectbox("Select a contact to delete", contacts_df['Name'].unique())

    if st.button("Delete Contact"):
        delete_contact(connection, selected_name)
        st.success(f"Contact {selected_name} deleted!")
        contacts_df = load_contacts(connection)  # Reload contacts

elif choice == 'View All Contacts':
    st.header("All Contacts")
    st.table(contacts_df)

elif choice == 'Search Contacts':
    st.header("Search Contacts")
    search_name = st.text_input("Enter name to search:")

    if st.button("Search"):
        result = contacts_df[contacts_df['Name'].str.contains(search_name, case=False)]
        if not result.empty:
            st.table(result)
        else:
            st.warning("No contact found with that name!")

# Close the database connection
if connection.is_connected():
    connection.close()
