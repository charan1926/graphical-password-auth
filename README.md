
# Project Title

Graphical Password Authentincation
Setup Guide
Copy & Paste the below commands step by step.

# clone repo
git clone https://github.com/charan1926/graphical-password-auth.git

# Navigate to the folder
cd graphical-password-auth

# Install Django into your system. 
pip install Django

# To make migrate run the below commands
python manage.py makemigrations
python manage.py migrate

# create admin account - Create a login to use on admin page
python manage.py createsuperuser

# Run Django server
python manage.py runserver

# this will run on localhost 127.0.0.0:8080

