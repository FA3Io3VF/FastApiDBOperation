# FastApiDBOperation
3 ways to improve DB management in FastApi


## First way

```python
async def create_users(db: AsyncSession):
    new_user = User(name="user1")
    db.add(new_user)
    await db.commit()
    await db.refresh(new_user)
    return new_user
```

## Second way

```python
async def create_user(db: AsyncSession, name: str) -> User:
    async with db.begin():
        new_user = User(name=name)
        db.add(new_user)
        await db.flush()
        await db.refresh(new_user)
    return new_user
```

## Third way

```python
async def create_user(db: AsyncSession, name: str) -> User:
    if not name:
        raise ValueError("Name must be a non-empty string")
    if not db.is_active:
        raise ValueError("Database is not connected")
    try:
        async with db.begin():
            new_user = User(name=name)
            db.add(new_user)
            try:
                await db.flush()
                if not new_user.id:
                    raise ValueError("User was not created in the database")
            except Exception as e:
                await db.rollback()
                logger.exception("Error flushing changes to database")
                raise ValueError(f"Error flushing changes to database: {e}")
            try:
                await db.refresh(new_user)
            except Exception as e:
                await db.rollback()
                logger.exception(f"Error refreshing user from database: {new_user.id}")
                raise ValueError(f"Error refreshing user from database: {e}")
        logger.info(f"User created with id {new_user.id}")
        return new_user
    except Exception as e:
        await db.rollback()
        logger.exception("Error creating user")
        raise e
```
