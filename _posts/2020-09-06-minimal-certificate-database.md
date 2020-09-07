---
layout: post
title: A Minimalist Certificate Database
---

There's a decentralized networked database system I'm working on (fog-db). It'd 
be really nice if it had some kind of authorization system, something that would 
let me identify anyone trying to access the database and decide if they're 
permitted to do so. In a decentralized system, that means, at minimum, using 
public-key cryptography and identifying users based on their public keys. 
However, I'd really like to do more than that.  I'd like the database to have 
more flexible rules, like permitting access to anyone whose key has been 
authorized by another key. If I want have keys authorizing other keys, that 
means I want...*certificates*.

Making Certificates
-------------------

If I mention certificates, the first thing programmers end up reaching for is 
X.509 certificates, which is a shame. X.509 is annoying to use. First defined in 
the late 1988, it makes a lot of assumptions about how certificates are 
generated and used. It can contain a lot of data too. There's a serial number. 
There's a subject name, an issuer name, their common names, policies, 
certificate revocation lists, certificate status protocol... the list goes on. 
You can bundle quite a lot into X.509. They're also heavily tied to existing 
notions on the internet, like servers, central authorities, and URLs.

Frankly, this is all too much, and it violates an absolute rule for my database 
system: **no URLs**. Actually, **no name authorities**. If I'm relying on DNS 
servers, or any name-to-identifier scheme, I've gone astray. They're not 
required in a decentralized system, and I don't want to bake them in.

So, I could bash X.509 all day, but the point is that's off the table. It's more 
than I need, anyway. I want a bare minimum. Bare minimums are good; there's less 
work for me to do, and therefore less for me to screw up. So what do I need, 
exactly?

Well, I said I want a key holder to be able to authorize another key holder. I 
probably want to be more specific than that. I might want to authorize specific 
things, maybe give people "roles", like "moderator", "member", "admin", etc. 
Perhaps I have more than one type of authorization to give out, too. Finally, I 
definitely want to give a start and end time to my authorization, to guarantee 
that my authorization isn't indefinite.

That last one is particularly important, by the way. Not recognizing how crucial 
it is led to X.509 on the internet having certificate revocation lists, then 
giving up on that and using the "Online Certificate Status Protocol", which is 
functionally having a second certificate that's valid for a smaller time period. 
The long history of X.509 has shown that the *only* way to make sure 
certificates become unusable and can be revoked is by making them valid for 
shorter time periods. Revocation through expiration is the only sure way.

Sorry, got caught up complaining about X.509 again. I'll stop after this, I 
promise. While I'm on it though, I recommend reading 
["Everything you Never Wanted to Know about PKI but were Forced to Find Out"][1],
by Peter Gutmann, if you want to find out what's screwy with the web's public 
key infrastructure. There's also [his slides on X.509][2], so you can get mad 
at X.509 specifically. My approach here won't address all of the fundamental 
issues he brings up, in part because this isn't remotely as ambitious as X.509 
is. Can't fail at something you don't try to do, right?

[1]: https://www.cs.auckland.ac.nz/~pgut001/pubs/pkitutorial.pdf
[2]: https://www.cs.auckland.ac.nz/~pgut001/tutorial/T2a_X509_Certs.pdf

Anyway... back to the task at hand. My wants can be boiled down to four things:

1. An issuer key can authorize a subject key.
2. The issuer assigns a specific value to the subject.
3. The issuer can assign more than one value to the subject.
4. There are specific start and end times.

Alright, 1 is easy. I want this certificate thing to be a document that the 
issuer signs using their private key, and the document must include both the 
issuer's public key and the subject's public key. For 2 & 3, I think the way to 
go here is having a *key-value* pair. I don't want to get caught up in having 
numbers and ranges and operations on my values, so both the key and value will 
be strings. Anything that checks certificates will do one operation: is the key 
X set to value Y? Finally, for 4, I just need to include start & end times in a 
format that is well-understood.

Well, wouldn't you know it, my [fog-pack][3] serialization format can cover 
all of that. Certificates will have just 5 fields: subject, key, value, start 
time, and end time. Signing the certificate in fog-pack means the issuer's 
public key is included as part of the signature.

[3]: https://crates.io/crates/fog-pack

Now, fog-pack supports [schema][4], so let's set up what that looks like:

[4]: https://docs.rs/fog-pack/0.1.1/fog_pack/struct.Schema.html

```json
{
  "req": {
    "subject": { "type": "Ident", "query": true },
    "key": { "type": "Str", "max_len": 255, "query": true },
    "value": { "type": "Str", "max_len": 255 },
    "start": { "type": "Time", "query": true, "ord": true },
    "end": { "type": "Time" }
  }
}
```

I limited the key and value strings to be no more than 255 characters, which is 
arbitrary but reasonable enough. These should be single words or phrases, not 
long sentences. So now we've got a certificate format that creates short 
certificates, like, for example:

```json
{
  "subject": "<Identity(Subject Key)>",
  "key": "forum role",
  "value": "moderator",
  "start": 1599409376,
  "end": 1600014176
}
```

Once signed, this certificate will be good from around noon on September 6, 
2020, to one week from then, and assigns the value "moderator" to the key "forum 
role" for the given subject.

What I like about this is each certificate asserts exactly one thing. If you 
want to assert more things, make more certificates! Sure, it doesn't allow for 
atomic changes to multiple key-value pairs, but I've yet to think of a case that 
absolutely requires that from a certificate system. Heck, about the only 
non-crypto thing people care about in website X.509 (unless you're a security 
researcher, maybe?) is one key: the DNS name. Atomic changes to key-value pairs 
just don't matter in the vast majority of cases.

Using Certificates
------------------

Alright, I've got a certificate format that's pretty small and simple. Now, how 
do I use them? Going back to the beginning, I wanted certificates because I want 
to have permission rules that let one key pair (the issuer) grant access to 
another key pair (the subject). There's actually two other things I'd like: to 
be able to chain those rules together, and to be able to require more than one 
issuer has created the same certificate. These additions let me create 
certificate chains, which are pretty important. For example, with a certificate 
chain, I can keep my "root" private key locked away in a big vault somewhere, and 
drag it out once every few months to sign certificates for my new day-to-day 
key pairs, which I in turn use to issue certificates to my users. Much more 
secure than keeping my most important key loaded up on a computer that might get 
hacked.

So, basic rules that can require multiple issuers, and chaining those rules. 
Let's go. When making a rule, I'll have a list of issuer public keys whose 
certificates I trust. Let's call those my "root" public keys. I'll then specify 
a rule: for a given subject key, if there is a certificate with key X set to 
value Y, and it is signed by one of the root keys, then that subject key is 
permitted access. I'll extend that by optionally requiring N certificates, where 
each one has a different issuer. I'll allow these rules to be chained (a *rule 
chain*): if a subject key meets one rule, they may act as an issuer for the next 
rule.  Finally, I'll allow more than one rule chain: if any of the specified 
rule chains are met, access is permitted. The structure of the rule set can also 
be expressed as a fog-pack schema, so let's do that:

```json
{
  "req": {
    "roots": {
      "type": "Array",
      "extra_items": {
        "type": "Ident"
      }
    },
    "chains": {
      "type": "Array",
      "comment": "Array of rule chains",
      "extra_items": {
        "type": "Array",
        "comment": "A single rule chain",
        "extra_items": {
          "type": "Obj",
          "comment": "A single certificate rule",
          "req": {
            "key": { "type": "Str", "max_len": 255 },
            "value": { "type": "Str", "max_len": 255 },
            "min_issuers": {
              "type": "Int",
              "min": 0,
              "max": 255
            }
          }
        }
      }
    }
  }
}
```

For each rule chain, the first rule uses the root keys as issuers, with any key 
as the subject. The second rule uses subject keys meeting the first rule as 
issuers, and allows any key to be the subject. And so on, until we run out of 
rules and the subject key is the one being checked by the rule. It's expected 
that these rules will be run in reverse, starting from the subject key being 
checked and moving upward in a breadth-first search.

This is all rather abstract. Let's do an example: I'm setting up a forum, and I 
want to allow all forum members to access my forum's posts. I'll make a set of 3 
root keys, and require that my forum admins be authorized by at least 2 of them. 
That way if I lose a root key, I can still operate (and maybe change over to a 
new set of root keys). I'll say that members are people who have been authorized 
by at least one admin. The resulting rule might look like:

```json
{
  "roots": [
    "<Identity(root key 0)>",
    "<Identity(root key 1)>",
    "<Identity(root key 2)>"
  ],
  "chains": [
    [
      { "key": "role", "value": "admin", "min_issuers": 2 },
      { "key": "role", "value": "member", "min_issuers": 1 }
    ]
  ]
}
```

Alright, not too painful to understand (I hope). But do I really even need this 
"role" thing? I made these root keys just for my forum. I can assign simple 
true-false properties instead: is a subject key a "admin", and is a subject 
key a "member". As long as a certificate exists and is signed, we'll say the 
property is true, and we don't even need a value anymore. (That's not to say 
key-value pairs aren't useful, just that we don't need them here.) I'll go one 
step further and allow members signed by either the root keys or any admin keys:

```json
{
  "roots": [
    "<Identity(root key 0)>",
    "<Identity(root key 1)>",
    "<Identity(root key 2)>"
  ],
  "chains": [
    [
      { "key": "admin", "value": "", "min_issuers": 2 },
      { "key": "member", "value": "", "min_issuers": 1 }
    ],
    [
      { "key": "member", "value": "", "min_issuers": 1 }
    ]
  ]
}
```

There we go. More complex rules can be built up if needed. One "weakness" here 
is that arbitrary-depth chains aren't allowed, but I think that's ok: I've yet 
to see a certificate chain in the wild deeper than 4, and the few bits of 
discussion I've found about certificate chaining mention up to 8. 8 might be a 
bit verbose in this rule language, but even that wouldn't be too bad.

So, now I've got a rule system for certificates. Cool. What am I going to use to 
check if a subject key meets these rules, given my set of available 
certificates?

The Certificate Database
------------------------

It's time for a database to hold all these certificates; a single place to store 
them for later rule-checking.

First, I don't want to hold onto all certificates indefinitely - they do have 
end times, after all. Second, I don't want a key-value pair to have more than 
one value for a key at a time, so I only want to have one certificate for each 
subject-key-issuer tuple. Third, that means I can replace certificates, so I 
want to store just the "newest" certificate, and keep it until the highest end 
time I've seen for that specific issuer-subject-key tuple.

This is easy to express in a key-value database with prefix lookups: each key in 
the database will be "subject-key-issuer", and the value will be the newest 
certificate plus the highest end time seen. When a new certificate is pushed to 
the database, we'll look up where it would go in the database, and compare 
against the current certificate and the end time. The certificate with the 
highest start time will be stored, and we'll store the maximum end time by 
looking at the new certificate's end time, comparing it to the currently stored 
end time, and storing whichever is highest. So, to summarize:

- key-value database
- Lookup by subject, then certificate key, then issuer
- Value holds both the certificate document and highest seen end time
- Certificates with matching subject-key-issuer tuples are compared, and the 
	newest one is stored. The highest end time seen for that tuple is recorded.

With the key-value database set up, we can periodically loop through and drop 
any certificates whose end time is older than the current time, because we know 
for certain that all matching certificates we've seen so far are expired. We can 
also use prefix lookup when evaluating our rules.

By keeping certificates seen in a local database, we have two choices for a 
response when asking it to evaluate our rules: it can return a one-time yes/no 
response, or it can return a token that will be dynamically updated anytime the 
database's stored certificates change or become expired.

While I haven't written a complete implementation of this yet, I did go ahead 
and define what the interface should look like in Rust. There's functions to add 
certificates, to look up certificates, and to verify a subject against a full 
certificate rule set (called a `CertRule` in the code):

```rust
trait CertDb {

  /// Add a certificate document to the database. Fails if the document is not a 
  /// valid, signed certificate. On success, returns true if the cert is now the 
  /// newest.
  fn add(&mut self, cert: Document) -> Result<bool, CertErr>;

  /// Add a set of certificate documents to the database. Fails if any of the 
  /// documents are not valid, signed certificates.
  fn add_vec(&mut self, certs: Vec<Document>) -> Result<(), CertErr>;
  
  /// Iterator over all certificates matching the given subject. Returns the 
  /// currently stored certificate and the highest end time seen for that 
  /// certificate.
  fn find_by_subject(&self, subject: Identity) 
    -> Box<dyn Iterator<Item = (Document, Timestamp)>;
  
  /// Iterator over all certificates matching the given issuer. Returns the 
  /// currently stored certificate and the highest end time seen for that 
  /// certificate.
  fn find_by_issuer(&self, issuer: Identity)
    -> Box<dyn Iterator<Item = (Document, Timestamp)>;
  
  /// Look up a certificate, given the subject, issuer, and key string. Returns 
  /// the currently stored certificate and the highest end time seen for that 
  /// certificate.
  fn find_tuple(&self, subject: Identity, issuer: Identity, key: &str)
    -> Option<(Document, Timestamp)>;
  
  /// Check to see if a subject meets the requirements of a certificate rule. 
  /// The returned token will be dynamically updated as the database changes.
  fn verify(&mut self, subject: Identity, rule: CertRule)
    -> Box<dyn CertToken>;
  
  /// Check to see if a subject meets the requirements of a certificate rule.  
  /// Checks only once, so the result is only valid at the exact point in time 
  /// when this function is called.
  fn verify_once(&self, subject: Identity, rule: CertRule) -> bool;
  
  /// Try to prove that a subject meets the requirements of a certificate rule. 
  /// If it does, all the certificates that were used to show the requirements 
  /// are met will be returned. By splitting up a `CertRule`, this function may 
  /// be used to dynamically explore the database.
  fn prove(&self, subject: Identity, rule: CertRule) -> Option<Vec<Document>>;
}

/// An authentication token. It indicates if a given subject Identity matches a 
/// certificate rule, according to the CertDb that issued it. It dynamically 
/// updates whenever CertDb receives new certificates. the CertDb `add` & 
/// `add_vec` functions are guaranteed to not return until all CertTokens have 
/// been appropriately updated.
trait CertToken {
  fn valid(&self) -> bool;
}
```

The rest of the interface (including what a `CertRule` struct looks like) will 
be in the fog-db crate, as I move forward with it.

Parting Thoughts
----------------

This simplified certificate system is good enough for my purposes, but leaves at 
least one enormous, glaring gap: from where do we fetch certificates?! Well, 
sorry, but a definitive answer is out of scope for this particular post. 
Roughly, I believe they will either be fetched in a stream of updates from the 
issuer, or they will be fetched on demand by asking remote peers to "prove" they 
meet a particular certificate rule. In fog-db, both of these styles can be done 
in a decentralized way, though the option to always contact a specific peer for 
certificate checking will remain.

That aside, hopefully this is a bit easier to deal with than X.509 and the full 
public key infrastructure that comes with it. A user will only need to depend on 
fog-pack, the fog-db trait & struct definitions, and a library implementing the 
certificate database itself. Bring your own network stack (or use more of 
fog-db).
