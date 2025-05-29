# Template of a System Design Interview

Your budget for interview is **35 minutes**. Interview are usually scheduled for
45 minutes. At Staff+ level you are expected 
to be proactive and drive the interview. The burden of proving that you are are
a *good* candidate lies with you at this level. 


## Functional Requirements (3 min)
1. Interview Starts with this. This is what of the problem. What is the system
    supposed to do ?
2. You should gather requirements **HERE**. Keep them crisp, simple and 
    achievable. 
3. Clarify problem and eliminate assumptions
4. Keep this conversational. The problem will be intentionally vague. It is your
    burden to drive the requirement gathering. No different than what happens at
    work.
5. Prioritize top 3 requirements.

## Non Functional Requirements (2 min)
1. Non-Functional requirements are the about the qualities of the system we 
    designing. This is the **crux** of the problem. Most caKeep these meaningful and in the context of the system we 
    are designing. Avoid vague statements. Pick 3 from below list. You are expected
    to **deep dive into at least 2** of these.
2. Checklist to follow for coming up with non-functional requirements,
    
    1. CAP Theorem. 
        
        **C is Single Copy Consistency**. Once data is written, all the nodes 
        should see the data at the same time. This is not same as the C in ACID.
        Most of time this is not really required. The places this is really required
        is in transactional systems like reservations, payments etc. 

        **A is Availability**. Node failures do not prevent survivors from operating.
        We need this all the time. 

        **P is Partition Tolerance**. We can never not choose partition tolerance, as
        it is not in our hand. 

        So we will end up with either CP or AP systems. There is no requirement that
        all of the system should be either CP or AP, parts of it can operate in different
        manners. It is entirely possible to get CP and AP for different parts of the 
        system at the same time. 

        **Think out loud** here. Show that you are able to think about the system
        and its trade offs. 

    2.  Scalability

        Ask about the scale of the system. Data to gather,
        
        1. DAU - Daily Active Users
        2. How many unique entities each user interacts with
        3. Kind of interaction, **Read vs Write**. This determines the 
            *Read vs Write* skew
        4. Peak rate of interaction, usually *2x* avg rate
        5. Average size of the entity
    
    3. Latency

        How much should be the end 2 end delay when a user is interacting with 
        entities ?
    
    4. Fault Tolerance

        How does the system handle failures ? 
    
    5. Environment Constraints 

        Low bandwidth, Limited resources, Networking issues 
    
    6. Durability 

        Can the system tolerate data loss ? 

    7. Security

        ACLs, Audit, etc.,
    
    You will usually pick 3 from the CAP, Scalability, Latency and Fault Tolerance.

## Entities and APIs (7 min)

### Entities (2 min)

Identify entities you are dealing with. Entities are usually persisted. Keep this 
brief and simple. You can always come back and update this as you find more
information. Entities should tie back to the functional requirements. 

### APIs (3-5 minutes)

High level contracts between the components of the system and its users. Most of
the time you would be using REST with HTTP. If you
are going to be using Websockets, you should specify the wire protocol. This should
tie back to the functional requirements.


## High Level Design (7-10 min)

Come up with an Architecture that satisfies the functional requirements using the
entities and APIs you have identities so far. Keep this simple and explain the data
flow. You will use building blocks to come up with the HLD. You should clearly 
demonstrate how the HLD satisfies the functional requirements.

## Capacity Estimation (<= 2 min)



## Low Level Design (10-15 min)

The goal to LLD is to extend the HLD to satisfy the non functional requirements. 
For scaling you will need to do capacity estimation to make design decisions. Deep dive
into at least 2 of the non functional requirements you have identified.  