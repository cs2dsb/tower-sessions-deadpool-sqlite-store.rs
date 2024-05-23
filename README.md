# deadpool-sqlite session store for `tower-sessions`

[![Crate](https://img.shields.io/crates/v/tower-sessions-deadpool-sqlite-store.svg)](https://crates.io/crates/tower-sessions-deadpool-sqlite-store)
[![Documentation](https://docs.rs/tower-sessions-deadpool-sqlite-store/badge.svg)](https://docs.rs/tower-sessions-deadpool-sqlite-store)

An implementation of `SessionStore` from [`tower-sessions`](https://github.com/maxcountryman/tower-sessions) that uses [`deadpool-sqlite`](https://github.com/bikeshedder/deadpool) as the backing store.

It currently uses [`serde_json`](https://github.com/serde-rs/json) for serializing the session because I wanted them to be human readable for debugging purposes but it could be adapted to use something more compact if performance is a concern.

## Usage

```
// Create the deadpool-sqlite database pool 
let pool = Config::new(args.sqlite_connection_string)
    .builder(Runtime::Tokio1)?
    // This is not necessary for the session store but I've left it in because it was hard to find
    // an example of using post_create
    .post_create(Hook::async_fn(|object, _| {
        Box::pin(async move {
            object
                .interact(|conn| db::configure_new_connection(conn))
                .await
                .map_err(AppError::from)?
                .map_err(AppError::from)?;
            Ok(())
        })
    }))
    .build()?;

// Create the session store
let session_store = DeadpoolSqliteStore::new(pool.clone());
// Call migrate to create the session table if it doesn't exist
session_store.migrate().await?;


axum::serve(
    ...
    ...
    .layer(
        // Pass the session_store to the session manager layer
        SessionManagerLayer::new(session_store)
            .with_secure(args.secure_sessions)
            .with_expiry(Expiry::OnInactivity(Duration::days(
                args.session_expiry_days,
            ))),
    ),
    ...
    ...
)
.await?;
```

## Disclaimer

This was created for my own usage in a hobby project so support may be spotty. Feel free to raise issues but don't depend on
it for any mission critical applications.