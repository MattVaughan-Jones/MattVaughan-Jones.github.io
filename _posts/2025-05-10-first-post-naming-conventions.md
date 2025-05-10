## Naming Conventions: My First (and Last?) Post

I wanted to make my first post easy to write, so here's shower thought... _Don't name things based on implementation context_.

Let's take a hypothetical: You use a CRM called The CRM Company (TCC). You build a bunch of models around it called including `tccCustomer` and `tccProduct`, which leverage an in-house library you've called `tcc`, and you've also built an internal API in it's own repository called `tccAPI`. Later you switch to Hubspot but you have all this tooling and cloud infrastructure which largely doesn't need to change but everything is still called `something-something-tcc-something`, and it's more hassle than it's worth to go through and update everything because everything is so tightly coupled, you can't change the name in one place without changing something in a million other places.

### Now the problems

Now we have some things named with "TCC", some named with "Hubspot" and (hopefully?) some named with "CRM".

- New people join the team who don't know what TCC is and have all kinds of problems.
- It's harder to search for documentation about CRM tooling.
- It's harder to find the repos where CRM-related stuff lives.
- It's harder to build a shared vocabulary.

All the while, some of the new stuff that gets built is called `tcc` for consistency and the problem grows instead of shrinking.

This all could have been avoided by naming everything with "CRM".

Don't name things based on implementation context.
