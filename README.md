# NLP-to-SQL-query-generater
from flask import Flask, render_template, request, jsonify, session, redirect, url_for
from dotenv import load_dotenv
import os
import pymysql
import google.generativeai as genai

# Load the Gemini-pro API key
app = Flask(__name__)
app.secret_key = '' #Add your Jemini api key

# Load environment variables
load_dotenv()
genai.configure(api_key="")  #Add your API key

def get_gemini_response(question, prompt):
    model = genai.GenerativeModel('gemini-pro')
    response = model.generate_content([f"{prompt}\n\n{question}"])
    return response.text.strip()

def get_db_schema(db_details):
    schema = {'tables': []}
    try:
        # Establish connection to the MySQL database
        conn = pymysql.connect(
            host=db_details['host'],
            user=db_details['user'],
            password=db_details['password'],
            database=db_details['database']
        )
        cur = conn.cursor()
        
        # Fetch table names
        cur.execute("SHOW TABLES;")
        tables = cur.fetchall()
        
        if not tables:
            return "No tables found in the database."
        
        for table in tables:
            table_name = table[0]
            schema['tables'].append(table_name)
            # Fetch column names for each table
            cur.execute(f"DESCRIBE {table_name};")
            columns = cur.fetchall()
            schema[table_name] = [col[0] for col in columns]  # Column names
        
        conn.close()
        return schema
    except Exception as e:
        return f"Error: {str(e)}"

#Generate SQL query using the Ggemini-pro
def generate_prompt(schema):
    prompt = "You are an expert in converting English questions to SQL queries. The SQL database has the following tables and columns:\n\n"
    
    for table, columns in schema.items():
        if table != 'tables':  # Skip 'tables' key
            prompt += f"Table: {table}\nColumns: {', '.join(columns)}\n\n"
    
    prompt += """Examples:

1. Question: How many entries of records are present?
   SQL: SELECT COUNT(*) FROM [table_name];

2. Question: Tell me all the students studying in Data Science class?
   SQL: SELECT * FROM [table_name] WHERE [column_name]='Data Science';

3. Question: What is the average mark of students in the Math class?
   SQL: SELECT AVG([column_name]) FROM [table_name] WHERE [column_name]='Math';

4. Question: List all students in Section A.
   SQL: SELECT * FROM [table_name] WHERE [column_name]='A';

5. Question: Find the names of students who scored more than 85.
   SQL: SELECT [column_name] FROM [table_name] WHERE [column_name] > 85;

Ensure the SQL code does not have backticks or the word 'sql' in the output."""

    return prompt


#Reades the generated SQL query.
def read_sql_query(sql, db_details):
    try:
        conn = pymysql.connect(
            host=db_details['host'],
            user=db_details['user'],
            password=db_details['password'],
            database=db_details['database']
        )
        cur = conn.cursor()
        cur.execute(sql)
        rows = cur.fetchall()
        columns = [desc[0] for desc in cur.description]
        conn.commit()
        conn.close()
        return columns, rows
    except Exception as e:
        return f"Error: {str(e)}", []

#Renders the home page.
@app.route('/')
def index():
    return render_template('index.html')

#Renders the page for query submission.
@app.route('/querypage')
def querypage():
    return render_template('querypage.html')

#Processes the database connection details provided by the user
@app.route('/submit_db_details', methods=['POST'])
def submit_db_details():
    host = request.form['host']
    user = request.form['user']
    password = request.form['password']
    database = request.form['database']
    db_details = {'host': host, 'user': user, 'password': password, 'database': database}

    schema = get_db_schema(db_details)
    if isinstance(schema, str):
        return jsonify({'success': False, 'message': schema})
    
    session['db_details'] = db_details
    prompt = generate_prompt(schema)
    session['prompt'] = prompt
    return jsonify({'success': True, 'tables': schema['tables']})

# it will save the database in a user system
@app.route('/save_db_details', methods=['POST'])
def save_db_details():
    save = request.form['save']
    if save == 'yes':
        session['save_db_details'] = True
        return jsonify({'success': True, 'message': 'Database details saved successfully!'})
    else:
        session.pop('db_details', None)
        return jsonify({'success': True, 'message': 'Database details were not saved.'})

@app.route('/submit_query', methods=['POST'])
def submit_query():
    question = request.form['question']
    prompt = session.get('prompt')
    db_details = session.get('db_details')
    
    if not prompt or not db_details:
        return jsonify({'success': False, 'message': 'Database details are missing. Please provide them first.'})
    
    gemini_response = get_gemini_response(question, prompt)
    sql_query = gemini_response  # Assuming the API returns the SQL query directly
    
    columns, query_results = read_sql_query(sql_query, db_details)
    if isinstance(query_results, str):
        return jsonify({'success': False, 'message': query_results})
    
    # Convert results to a format that can be easily used in JavaScript
    query_results_json = [
        {columns[i]: row[i] for i in range(len(columns))}
        for row in query_results
    ]
    
    return jsonify({'success': True, 'columns': columns, 'data': query_results_json})

if __name__ == '__main__':
    app.run(debug=True, host='0.0.0.0', port=8080)


# Test Cases

# 1. Accuracy Testing
# - Verify that the SQL queries generated by the AI model return the expected results.
# - Compare the output of the generated SQL queries with manually written queries for the same questions.

# 2. Loss Analysis
# - Monitor the performance of the AI model over time.
# - Analyze the loss during training (if applicable) to ensure the model is improving.

# 3. Functionality Testing (Text Mode)
# - Test various natural language questions to ensure the application correctly interprets and converts them to SQL queries.
# - Include edge cases and ambiguous questions to evaluate robustness.

# 4. Comparison with Standard Models (RNN, Decision Tree)
# - Implement standard models (like RNN or Decision Tree) for SQL query generation.
# - Compare the performance, accuracy, and response time of the Gemini model against these standard models.

# 5. User Query Variability Testing
# - Test the application with a wide range of user queries to ensure it can handle different phrasings and synonyms.
# - Evaluate how well the application generalizes across various question formats.

# 6. Application Usability Testing
# - Conduct user testing sessions to gather feedback on the application's interface and user experience.
# - Assess how intuitive the application is for users with varying levels of technical expertise.

# 7. Real-Time Data Testing
# - Test the application with real-time data updates in the database to ensure it can handle dynamic data.
# - Verify that the application responds correctly to queries based on the most current data.
