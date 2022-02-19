# Introduction

In this File the project code is analysed.

# Code Details

## Predicate: prob_calculation

The predicate prob_calculation consists of 6 cases, in which, depending on the date of contact between a none infected person and an infected person, attributes one of the following values: [0.01; 0.1; 0.3; 0.5; 0.7;
0.9]. The values ​​are attributed according to the reported contagion probability graph
previously (days are expressed in milliseconds).

```prolog
prob_calculation(X,Y,Z,0.01):-
      X>=Y-(Z/2), X<Y-1123200000
      ; X>=Y+1123200000, X=<Y+(Z/2).
prob_calculation(X,Y,_,0.1):-
      X>=Y-1123200000, X<Y-1036800000;
      X>Y+1036800000, X=<Y+1123200000.
prob_calculation(X,Y,_,0.3):-
      X>=Y-1036800000, X<Y-864000000;
       X>Y+864000000, X=<Y+1036800000.
prob_calculation(X,Y,_,0.5):-
      X>=Y-864000000, X<Y-604800000;
      X>Y+604800000, X=<Y+864000000.
prob_calculation(X,Y,_,0.7):-
      X>=Y-604800000, X<Y-289200000;
      X>Y+289200000, X=<Y+604800000.
prob_calculation(X,Y,_,0.9):-
      X>=Y-289200000, X=<Y+289200000.
```

## Predicate: calculate_distance

Predicate that given two geographic coordinates, expressed in latitude and longitude, calculates the
distance.
"Haversine" formula is used:

$$a ={sin^2(\frac{\Delta \varphi}{2})+cos\varphi_1*cos\varphi_2*sin^2(\frac{\Delta \gamma}{2})}$$

$$c= 2*\arctan2(\sqrt{a},\sqrt{1-a}) $$

$$d=R*c$$

Where:

- 𝜑 is the latitude
- 𝛾 is the longitude
- 𝑅 is the earth's radius equivalent to 6371000 meters.

Before applying the formula the latitude and longitude measurements are converted to
radians by means of the predicate to_radians.

```prolog
calculate_distance(Lat1,Lat2,Lon1,Lon2,Distance):-
     to_radians(Lat1,Latr1),
     to_radians(Lat2,Latr2),
     Delta_phi is Lat2-Lat1,
     Delta_lambda is Lon2-Lon1,
     to_radians(Delta_phi,Delta_phi_r),
     to_radians(Delta_lambda,Delta_lambda_r),
     Earth_Radius is 6371000,
     A is ((sin(Delta_phi_r*0.5)*sin(Delta_phi_r*0.5)+
           cos(Latr1)*cos(Latr2)*sin(Delta_lambda_r*0.5)
           *sin(Delta_lambda_r*0.5))),
     C is (2*atan2(sqrt(A),sqrt(1-A))),
     Distance is (Earth_Radius*C),
     Distance<5.
```

## Predicate: calculate_contact_time

This predicate checks if:

- person1 arrival time < person2 leave time
- person2 arrival time < person1 leave time

If these conditions are true then we can calculate the period of time that the two people are in contact as:

take the minimum between the times in which the two people leave the place and
subtract from it the maximum between the arrival times of the two people.

```prolog
calculate_contact_time(T1_start,T1_end,T2_start,T2_end,Period,Tstart):-
    T1_start < T2_end,
    T2_start < T1_end,
    Tstart is max(T1_start,T2_start),
    Tend is min(T1_end,T2_end),
    Period is Tend-Tstart.

```

## Predicate: is_contact

this predicate is used to calculate if person1 and person2 are at the same place at the same time.--> Person1 and Person2 are been in contact.

```prolog
is_contact(Person1,Person2,Tstart,Distance,Place_ID,Period):-
    a(Person1,T1_start,Lat1,Lon1,T1_end,Place_ID),
    a(Person2,T2_start,Lat2,Lon2,T2_end,Place_ID),
     Person1 \= Person2,
    \+positive(Person2,_),
      calculate_distance(Lat1,Lat2,Lon1,Lon2,Distance),
    calculate_contact_time(T1_start,T1_end,T2_start,T2_end,Period,Tstart)
```

## Predicate: is_infected

The is_infected predicate controls whether Person1 infects person2 or not, via 3
necessary conditions for the contagion to occur:

- The contact between the two people must take place during the period that Person1 is
  contagious;
- The contact period is greater than the minimum exposure time in the same
  place for contagion to occur;
- The distance between the two people is less than the maximum distance for there to be the
  infection.

```prolog
is_infected(Person1,(Person2,Contact_Date,Place_ID,Person1,Probability)):-
    positive(Person1,Tampon_Date),
    is_contact(Person1,Person2,Contact_Date,Distance,Place_ID,Period),
    input_data(Infection_Period,_,Min_Exposure_Time,Min_Distance),
    Date is Tampon_Date-(Infection_Period/2),
    Contact_Date >= Date,
    Distance < Min_Exposure_Time,
    Period > Min_Distance,
    prob_calculation(Contact_Date,Tampon_Date,Infection_Period,Probability),
    insert_infected(Person2,Contact_Date,Probability).

```

```prolog
is_infected(Person1,(Person2,Contact_Date,Place_ID,Person1,New_Probability)):-
    infected(Person1,Infection_Date,Prev_Probability),
    is_contact(Person1,Person2,Contact_Date,Distance,Place_ID,Period),
    input_data(Periodo_Contagio,Infection_Period_Start,Min_Exposure_Time,Min_Distance),
    Date is Infection_Date+Infection_Period_Start,
    Date =< Contact_Date,
    Data2 is Date+(Periodo_Contagio/2),
    Distance < Min_Distance,
    Period > Min_Exposure_Time,
    prob_calculation(Contact_Date,Data2,Periodo_Contagio,Probability),
    New_Probability is Prev_Probability*Probability,
    insert_infected(Person2,Contact_Date,New_Probability).

```

## Predicate: are_infected_list

Depth-First-Search Algorithm.

Take Person1, check if Person2 has been infected by Person1:

- if yes then insert Person2 in a List and repeat the algorithm for Person2;
- otherwise discard Person2 and repeat the algorithm for Person1.

```prolog
are_infected_list(Person1,[(Person2,Tstart,Place_ID,Person1,Probability)|Others]):-
        is_infected(Person1,(Person2,Tstart,Place_ID,Person1,Probability)),
                are_infected_list(Person2,Others).

are_infected_list(Person1,[]):-
    \+is_infected(Person1,_),!.
```

## Predicate: find_all_infected

The predicate find_all_infected takes as input the list of positives and for each element of the list
searches the possible lists of infected people and inserts them on List2.

```prolog
find_all_infected([H|T],[List2|T1]):-
    findall(List,(are_infected_list(H,List)),List2),
    find_all_infected(T,T1).

find_all_infected([],[]):-!.
```

## Main Program

Main program:

- At the beginning it cleans the dynamic database of facts of the infected people and data_input;
- It requests the input parameters and saves them in a dynamically created fact
  (data_input / 4).

- Find all the positive people saved in the database and put them in a list;
- Calls the predicate find_all_infected and returns the list of the infected people;
- Use the flatten predicate to transform the linked list (i.e. a list of
  lists of lists of records) in a normal list of records;
- The remove_duplicate predicate eventually deletes the records that are repeated,
  inserting them in the final list (Infected_List);
- Print on screen
- Clean up the dynamic database

```prolog
main:-
    retractall(infected(_,_,_)),
    retractall(input_data(_,_,_,_)),

    write("Insert the period in which a person can be contagious (in milliseconds):  "),nl,
    read(Contagious_Period),
    write("Insert incubation period (in milliseconds):  "),nl,
    read(Contagious_Period_Start),
    write("Insert minimum exposure time (in millisecondi):  "),nl,
    read(Min_Time_Exposure),
    write("Insert maximum distance(in metri):  "),nl,
    read(Dm),

    assert(input_data(Contagious_Period,Contagious_Period_Start,Min_Time_Exposure,Dm)),
    findall(Person1,positive(Person1,_),Positive_List),
    find_all_infected(Positive_List,All_Infected_List),
    flatten(All_Infected_List,Flatten_Infected_List),
    remove_duplicates(Flatten_Infected_List,Infected_List),

    nl, write("Person"),write("    "),write("Date"),
    write("                  "),
    write("Pace ID"), write("                "),
    write("Infected By"),  write("                "),
    write("Infection Probability"),nl,nl,

    print_all(Infected_List),
    retractall(infected(_,_,_)).

```
