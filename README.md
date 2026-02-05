# CVE-2025-64459 - Django SQL Injection Risk
#### Video: URL GOES HERE

## Introduction:

Hello. This is my submission for the CS50 Cybersecurity final project.
To get some housekeeping out of the way, my name is Darrell Kiely. I'm located in Osaka Japan, and I go by the username Eptalin on both GitHub and EdX. I'm recording this video on DATE.

The topic of today's video is CVE-2025-64459, a critical SQL injection vulnerability discovered in Django, which was published on the 5th of November at the tail end of last year, 2025.

### Selection

Before CS50 Cybersecurity, I had already completed a number of other CS50 units, including CS50x, CS50 SQL, and CS50 Web. I really enjoyed SQL, and this course covered injection attacks, so I thought it would be interesting to see if there had actually been any SQL injection attacks recently. 

Initially, I naiively thought my chances were slim, and I was expecting to maybe find something like a vibe-coded web app, where the creator unwittingly plugged user input directly into raw SQL queries; either trusting the AI too much, or simply not being aware of the security risk.

So I was extremely surprised to find that a SQL injection vulnerability was discovered in the Django web framework. SQL injection protections are one of the framework's selling points, and you interact with the database using Python, not raw SQL queries, so I was left wondering how it was even possible.

So you know, I had to know more, and after some research, now I do. So let's get into it.

## What is it?

First, what's the vulnerability?
CVE-2025-64459 is a potential SQL injection vulnerability for the Django web framework, which affects versions 5.2, 5.1 and 4.2. 
It has a CVSS 3.1 severity rating of 9.1 out of 10, marking it a 'critical' vulnerability. More on what that means a bit later. But first, let's look at this vulnerability, because it's a little unique. 

A SQL injection attack is a kind of code injection attack where an attacker inserts malicious SQL queries into something like an entry field of a web application. The harm is done when a web application takes that user input and uses it in their SQL queries without sanitising the input first. 

But as I mentioned before, when using Django you don't write raw SQL queries.
When a form is submitted to Django, the user input is fed into a purely Python statement. Behind the scenes, Django then queries the database for you, passing it the user input in a safe way, akin to using prepared statements.

So what went wrong? 
While the CVE report calls it a "potential SQL injection", it doesn't actually use any SQL syntax at all. It's really exploiting Django's QuerySet methods, like .filter(), .get() and .exclude(), which is what it uses to generate SQL queries.

### Example

Let's have a look at a fairly common, yet dangerous example. Take the following function:
```
def search_users(request):
    filter = requests.GET.dict()
    users = User.objects.filter(**filters)
    return users
```
We have a function named search_users which takes a HTTP request as input.
It then dynamically creates a list of filters based on whatever was included in that dictionary using dictionary expansion, this ** notation, and searches the database for users who meet the given conditions. 
For example, if the dictionary contains username=Eptalin, it will filter for users with that username.

Now, how could an adversary abuse this. Take the following HTTP request, [below]. 
```
GET /api/users?username='admin'&is_superuser=FALSE&_connector=OR
```
It's a GET request, so everything is visible in the URL, and we can see it's going to the users API route, and contains 3 things that will be plugged into the filters.
The username is 'admin', the user is NOT a superuser, and the connector in the query should be OR. And you may already see where this is going.

Ideally, Django would generate the following SQL query, [below], which should of course protect the database and return zero results.
```
WHERE username='admin' AND is_superuser=FALSE;
```

But by allowing the HTTP request to dictate the connector OR instead of AND, we get this query, [below]. This returns the admin, and all non-superusers, too.
```
WHERE username='admin' OR is_superuser=FALSE;
```

So without any credentials the adversary was able to get access to all users, including the admin itself. And depending on the API route and HTTP request, an adversary could do more than just access data. Under the right conditions, they could perform other CRUD operations, too, adding changing or deleting data.

### Severity

So onto that severity score from earlier.
The Common Vulnerability Scoring System (CVSS) scores severity in a range of 0-10 denoting the potential for harm, with 
0 being None
0.1 ~ 3.99 being Low
4 ~ 6.9 being Medium
7 ~ 8.9 being High, and
9 ~ 10 being Critical

This Django vulnerability scored 9.1 out of 10, a Critical Vulnerability. It earned that score by assessing certain criteria. First, the Exploitability metrics:

The Attack Vector:
Does the attacker need to be in physical contact with the server, on the same local network, or can they attack remotely over the internet.
In our case, it's remotely exploitable, the most dangerous.

The Attack Complexity:
Basically, how hard is it. Is it like Mission Impossible where they need to attack at a specific location in a specific way within a specific time window. Or can you expect repeatable success without any major prep work.
In this case, it was low complexity. You just send a malicious HTTP request.

The Privileges Required:
Who can exploit the vulnerability. Do you need to be a superuser, like an admin to do it? Can a regular user do it? Or can anyone at all do it even without an account.
Like the example we looked at before, absolutely anyone can send the malicious HTTP request and gain access.

Next, User Interaction:
Does a user need to do something, like click a suspicious link, in order for the system to be exploited.
In this case no. An attacker can gain access to everything without a user doing anything.

Scope:
This is about whether exploitation crosses security boundaries into a different authority. In this case, the scope is unchanged: although the impact is severe, the attacker remains within the Django application’s security context. They don't gain control over external systems or higher-privilege domains.

And now the Impact Metrics:

Confidentiality Impact:
Does the vulnerability allow access to restricted information. 
No surprise we got the highest score here. All information is divulged to the attacker, and the potential to do harm with that information is great.

Integrity Impact:
This is about an attackers ability to change files or data. 
This Django vulnerability potentially allows full CRUD capabilities, which is a total loss of integrity.

and finally, Availability Impact:
Basically, does the exploitation reduce the system performance or stop it from running, preventing users from using it.
While one could argue the deletion or corruption of all data in the database would cause disruptions to the service, that impact is already accounted for in the Integrity metric.
This Django vulnerability is judged to have no impact on the availability of the service.

Due to the ease at which an attacker can exploit the vulnerability, and the harm they can do, it's safe to say it earned that Critical label.


## Remedy (Django update and recommendations)

As is standard, the Django team only publicly disclosed this vulnerability once they'd released updates to the affected versions.

So the first, most-important step is to immediately update if you're using one of the affected versions. They add a couple of layers of protection: 
QuerySet validation let us disallow certain filters, like _connector or _negated, and 
Q Object validation lets us set a list of allowable connectors directly.

In addition to these, there are also some tips and best practices we can implement to avoid potential exploits like this in the future:
Don't pass request.GET.dict(), request.POST.dict(), etc directly to QuerySet methods.
Use Django Forms to validate all user input.
Implement parameter whitelisting for filter endpoints.
Use explicit field mapping rather than dictionary expansion.


## Avoiding SQL injection attacks more generally
{ no time for this }


## Closing

Thanks for watching this far. I had a great time looking into this. While I learned of a cool technique to make my own endpoints more dynamic, I very quickly learned why I probably shouldn't. At least, not without taking precautions to avoid misuse, and implementing tests to make sure they work.
Thank you also for introducing me to the CVE records. I'm definitely going to add researching CVE records to my study rotation.
And lastly and most importantly, thank you to David Malan and the team for putting these materials together and releasing them to the world. I greatly appreciate it.

Thank you again. This was CS50 Cybersecurity.


## Sources:
CVE Records Database: https://www.cve.org/CVERecord?id=CVE-2025-64459
National Vulnerability Database: https://nvd.nist.gov/vuln/detail/CVE-2025-64459
EndorLabs: https://www.endorlabs.com/learn/critical-sql-injection-vulnerability-in-django-cve-2025-64459