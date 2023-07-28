# Building-a-Bike-Rental-Shop

In this project, I will build an interactive Bash program that stores rental information for bike rental shop using PostgreSQL.  

## Step 1: Create Docker-compose file

You will use docker compose to create a container docker for Postgres.  

Pls create a file `docker-compose.yaml` and a folder `bikes_data`.    

File `docker-compose.yaml`    
```
services:
  pgdatabase:
    image: postgres:13
    environment:
      - POSTGRES_USER=root
      - POSTGRES_PASSWORD=root
      - POSTGRES_DB=bikes
    volumes:
      - "./bikes_data:/var/lib/postgresql/data:rw"
    ports:
      - "5432:5432"
```

To start a postgres instance, run this command:  
`sudo docker-compose up -d`

**Note:** If you want to stop that docker compose, pls enter this command: `sudo docker-compose down`  

Ensure that the .pgpass file is properly set up to avoid any password prompts. If the .pgpass file doesn't exist, create it in your home directory and set the appropriate permissions:

```
touch ~/.pgpass
chmod 600 ~/.pgpass
```

Open the .pgpass file in a text editor and add the following line with the appropriate values for your PostgreSQL server:

```
localhost:5432:bikes:root:your_password_here
``` 

To log in to PostgreSQL with psql. Do that by entering this command in your terminal:

```
psql -h <hostname> -p <port> -U <username> -d <database>
```

## Step 2: Create table in DB

Create 3 tables `bikes`, `customers` and `rentals` for DB like the below:

```
CREATE TABLE bikes (
    bike_id SERIAL PRIMARY KEY,
    type VARCHAR(50) NOT NULL,
    size INT NOT NULL,
    available BOOLEAN NOT NULL DEFAULT TRUE
);
```

```
CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    name VARCHAR(40) NOT NULL,
    phone VARCHAR(15) NOT NULL UNIQUE
);
```

```
CREATE TABLE rentals (
    rental_id SERIAL PRIMARY KEY,
    customer_id INT NOT NULL REFERENCES customers(customer_id),
    bike_id INT NOT NULL REFERENCES bikes(bike_id),
    date_rented DATE NOT NULL DEFAULT NOW(),
    date_returned DATE
);
```

Next, insert some data to table `bikes`

```
INSERT INTO public.bikes (type, size) VALUES 
('Mountain', '28'),
('Mountain', '29'),
('Mountain', '27'),
('Road', '27'),
('Road', '28'),
('Road', '29'),
('BMX', '19'),
('BMX', '20'),
('BMX', '21');
```

## Step 3: Create Bash file to store rental information for bike rental shop

And then, you will create the Bash script file `bike_shop.sh`. Ensure the script has execution permission: 

```
chmod +x bike_shop.sh
```

File `bike_shop.sh`  

```
#!/bin/bash

# Set PGPASSFILE environment variable to point to the .pgpass file
export PGPASSFILE=/home/sang/.pgpass

PSQL="psql -h localhost -p 5432 -U root -d bikes --no-align --tuples-only -c"

echo -e "\n~~~~~ Bike Rental Shop ~~~~~\n"

MAIN_MENU() {
  if [[ $1 ]]
  then
    echo -e "\n$1"
  fi

  echo "How may I help you?" 
  echo -e "\n1. Rent a bike\n2. Return a bike\n3. Exit"
  read MAIN_MENU_SELECTION

  case $MAIN_MENU_SELECTION in
    1) RENT_MENU ;;
    2) RETURN_MENU ;;
    3) EXIT ;;
    *) MAIN_MENU "Please enter a valid option." ;;
  esac
}

RENT_MENU() {
  # get available bikes
  AVAILABLE_BIKES=$($PSQL "SELECT bike_id, type, size FROM bikes WHERE available = true ORDER BY bike_id")
  # if no bikes available
  if [[ -z $AVAILABLE_BIKES ]]
  then
    # send to main menu
    MAIN_MENU "Sorry, we don't have any bikes available right now."
  else
    # display available bikes
    echo -e "\nHere are the bikes we have available:"
    echo "$AVAILABLE_BIKES" | while IFS='|' read -r BIKE_ID TYPE SIZE
    do
      echo "$BIKE_ID) $SIZE\" $TYPE Bike"
    done

    # ask for bike to rent
    echo -e "\nWhich one would you like to rent?"
    read BIKE_ID_TO_RENT

    # if input is not a number
    if [[ ! $BIKE_ID_TO_RENT =~ ^[0-9]+$ ]]
    then
      # send to main menu
      MAIN_MENU "That is not a valid bike number."
    else
      # get bike availability
      BIKE_AVAILABILITY=$($PSQL "SELECT available FROM bikes WHERE bike_id = $BIKE_ID_TO_RENT AND available = true")

      # if not available
      if [[ -z $BIKE_AVAILABILITY ]]
      then
        # send to main menu
        MAIN_MENU "That bike is not available."
      else
        # get customer info
        echo -e "\nWhat's your phone number?"
        read PHONE_NUMBER

        CUSTOMER_NAME=$($PSQL "SELECT name FROM customers WHERE phone = '$PHONE_NUMBER'")

        # if customer doesn't exist
        if [[ -z $CUSTOMER_NAME ]]
        then
          # get new customer name
          echo -e "\nWhat's your name?"
          read CUSTOMER_NAME

          # insert new customer
          INSERT_CUSTOMER_RESULT=$($PSQL "INSERT INTO customers(name, phone) VALUES('$CUSTOMER_NAME', '$PHONE_NUMBER')") 
        fi

        # get customer_id
        CUSTOMER_ID=$($PSQL "SELECT customer_id FROM customers WHERE phone='$PHONE_NUMBER'")

        # insert bike rental
        INSERT_RENTAL_RESULT=$($PSQL "INSERT INTO rentals(customer_id, bike_id) VALUES($CUSTOMER_ID, $BIKE_ID_TO_RENT)")

        # set bike availability to false
        SET_TO_FALSE_RESULT=$($PSQL "UPDATE bikes SET available = false WHERE bike_id = $BIKE_ID_TO_RENT")

        # get bike info
        BIKE_INFO=$($PSQL "SELECT size, type FROM bikes WHERE bike_id = $BIKE_ID_TO_RENT")
        echo $BIKE_INFO 
        BIKE_INFO_FORMATTED=$(echo $BIKE_INFO | sed 's/|/" /g')
        
        # send to main menu
        MAIN_MENU "I have put you down for the $BIKE_INFO_FORMATTED Bike, $(echo $CUSTOMER_NAME | sed -r 's/^ *| *$//g')."
      fi
    fi
  fi
}

RETURN_MENU() {
  # get customer info
  echo -e "\nWhat's your phone number?"
  read PHONE_NUMBER

  CUSTOMER_ID=$($PSQL "SELECT customer_id FROM customers WHERE phone = '$PHONE_NUMBER'")

  # if not found
  if [[ -z $CUSTOMER_ID  ]]
  then
    # send to main menu
    MAIN_MENU "I could not find a record for that phone number."
  else
    # get customer's rentals
    CUSTOMER_RENTALS=$($PSQL "SELECT bike_id, type, size FROM bikes INNER JOIN rentals USING(bike_id) INNER JOIN customers USING(customer_id) WHERE phone = '$PHONE_NUMBER' AND date_returned IS NULL ORDER BY bike_id")

    # if no rentals
    if [[ -z $CUSTOMER_RENTALS  ]]
    then
      # send to main menu
      MAIN_MENU "You do not have any bikes rented."
    else
      # display rented bikes
      echo -e "\nHere are your rentals:"
      echo "$CUSTOMER_RENTALS" | while IFS='|' read -r BIKE_ID TYPE SIZE
      do
        echo "$BIKE_ID) $SIZE\" $TYPE Bike"
      done

      # ask for bike to return
      echo -e "\nWhich one would you like to return?"
      read BIKE_ID_TO_RETURN

      # if not a number
      if [[ ! $BIKE_ID_TO_RETURN =~ ^[0-9]+$ ]]
      then
        # send to main menu
        MAIN_MENU "That is not a valid bike number."
      else
        # check if input is rented
        RENTAL_ID=$($PSQL "SELECT rental_id FROM rentals INNER JOIN customers USING(customer_id) WHERE phone = '$PHONE_NUMBER' AND bike_id = $BIKE_ID_TO_RETURN AND date_returned IS NULL")

        # if input not rented
        if [[ -z $RENTAL_ID ]]
        then
          # send to main menu
          MAIN_MENU "You do not have that bike rented."
        else
          # update date_returned
          RETURN_BIKE_RESULT=$($PSQL "UPDATE rentals SET date_returned = NOW() WHERE rental_id = $RENTAL_ID")
          
          # set bike availability to true
          SET_TO_TRUE_RESULT=$($PSQL "UPDATE bikes SET available = true WHERE bike_id = $BIKE_ID_TO_RETURN")
          
          # send to main menu
          MAIN_MENU "Thank you for returning your bike."
        fi
      fi
    fi
  fi
}

EXIT() {
  echo -e "\nThank you for stopping in.\n"
}

MAIN_MENU

```

Execute that Bash file: `./bike_shop.sh`. Feel free to play around, rent and return some bikes.
This is the result:

```
$ ./bike-shop.sh 

~~~~~ Bike Rental Shop ~~~~~

How may I help you?

1. Rent a bike
2. Return a bike
3. Exit
1

Here are the bikes we have available:
1) 28" Mountain Bike
2) 29" Mountain Bike
3) 27" Mountain Bike
4) 27" Road Bike
5) 28" Road Bike
6) 29" Road Bike
7) 19" BMX Bike
8) 20" BMX Bike
9) 21" BMX Bike

Which one would you like to rent?
1

What's your phone number?
123

What's your name?
Sang
28|Mountain

I have put you down for the 28" Mountain Bike, Sang.
How may I help you?

1. Rent a bike
2. Return a bike
3. Exit
1

Here are the bikes we have available:
2) 29" Mountain Bike
3) 27" Mountain Bike
4) 27" Road Bike
5) 28" Road Bike
6) 29" Road Bike
7) 19" BMX Bike
8) 20" BMX Bike
9) 21" BMX Bike

Which one would you like to rent?
4

What's your phone number?
456

What's your name?
An
27|Road

I have put you down for the 27" Road Bike, An.
How may I help you?

1. Rent a bike
2. Return a bike
3. Exit
2

What's your phone number?
123

Here are your rentals:
1) 28" Mountain Bike

Which one would you like to return?
1

Thank you for returning your bike.
How may I help you?

1. Rent a bike
2. Return a bike
3. Exit
3

Thank you for stopping in.

```

```
bikes=# select * from bikes;
 bike_id |   type   | size | available 
---------+----------+------+-----------
       2 | Mountain |   29 | t
       3 | Mountain |   27 | t
       5 | Road     |   28 | t
       6 | Road     |   29 | t
       7 | BMX      |   19 | t
       8 | BMX      |   20 | t
       9 | BMX      |   21 | t
       4 | Road     |   27 | f
       1 | Mountain |   28 | t
(9 rows)

bikes=# select * from customers;
 customer_id | name | phone 
-------------+------+-------
           1 | s    | s
           2 | Sang | 1
           3 | 2    | 2
           4 | 4    | 4
           5 | 6    | 6
           6 | 7    | 7
           7 | 8    | 8
           8 | 9    | 9
           9 | Sang | 123
          10 | An   | 456
(10 rows)

bikes=# select * from rentals;
 rental_id | customer_id | bike_id | date_rented | date_returned 
-----------+-------------+---------+-------------+---------------
         2 |           2 |       2 | 2023-07-28  | 2023-07-28
         3 |           3 |       3 | 2023-07-28  | 2023-07-28
         4 |           4 |       4 | 2023-07-28  | 2023-07-28
         5 |           4 |       5 | 2023-07-28  | 2023-07-28
         6 |           5 |       6 | 2023-07-28  | 2023-07-28
         7 |           6 |       7 | 2023-07-28  | 2023-07-28
         8 |           7 |       8 | 2023-07-28  | 2023-07-28
         9 |           8 |       9 | 2023-07-28  | 2023-07-28
         1 |           1 |       1 | 2023-07-28  | 2023-07-28
        11 |          10 |       4 | 2023-07-28  | 
        10 |           9 |       1 | 2023-07-28  | 2023-07-28
(11 rows)

```