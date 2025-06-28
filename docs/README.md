# go-site-builder
GO+HTMX+TEMPL website builder

This plan follows modern Go web application patterns and incorporates the research findings about onion architecture patterns for maintaining clean separation of concerns and multi-tenant data isolation strategies.

## Key Architecture Decisions Based on Research:

1. **THGM Stack**: Following the modern HTMX + TEMPL + Go pattern that's gaining popularity for server-side rendered applications

2. **Multi-tenancy Strategy**: Implementing database-per-tenant isolation for data privacy protection which scales better than shared databases for your use case

3. **Minimal Dependencies**: The plan uses only essential Go packages, avoiding heavy frameworks while maintaining functionality

4. **Nix Integration**: Structured for easy deployment and reproducibility across different environments

## Real-world Inspiration:

The architecture draws from successful SaaS platforms like:
- **Webflow**: Visual website builder with multi-tenancy
- **Ghost**: Self-hostable publishing platform
- **Supabase**: Open-source alternative to Firebase
- **Plausible Analytics**: Lightweight, self-hostable analytics

## Key Technical Benefits:

1. **Lightweight**: Single binary deployment with embedded assets
2. **Fast**: Server-side rendering with HTMX for smooth interactions
3. **Scalable**: Database-per-tenant allows horizontal scaling
4. **Secure**: Proper tenant isolation and minimal attack surface
5. **Maintainable**: Clean architecture with dependency inversion

The workplan includes detailed code examples, build automation, testing strategies, and deployment configurations. Each phase builds incrementally toward a production-ready system.

## Sample template repos we can use pieces of / repurpose 

* [**Go/Echo+Templ+Htmx:**](https://github.com/emarifer/go-echo-templ-htmx) Full stack application using Golang's Echo framework & Templ templating language with user session management + CRUD to a SQLite database (To Do List) and HTMX in the frontend

* [GO+TEMPL+HTMX fullstack app](https://dev.to/hackmamba/how-to-build-a-fullstack-application-with-go-templ-and-htmx-4444)

* [go data framework for saas(multi-tenancy)](https://github.com/go-saas/saas)

* [GO+HTMX basic webapp](https://dev.to/calvinmclean/how-to-build-a-web-application-with-htmx-and-go-3183)