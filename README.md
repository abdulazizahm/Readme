# Convert List To DataTable
public static DataTable ConvertToDataTable<T>(IList<T> data)
{
    PropertyDescriptorCollection properties =
       TypeDescriptor.GetProperties(typeof(T));
    DataTable table = new DataTable();
    foreach (PropertyDescriptor prop in properties)
        table.Columns.Add(prop.Name, Nullable.GetUnderlyingType(prop.PropertyType) ?? prop.PropertyType);
    foreach (T item in data)
    {
        DataRow row = table.NewRow();
        foreach (PropertyDescriptor prop in properties)
            row[prop.Name] = prop.GetValue(item) ?? DBNull.Value;
        table.Rows.Add(row);
    }
    return table;

}
# Repository pattern
# IRepository
   public interface IRepository<T, TKey>
   {
       IQueryable<T> GetAll(Expression<Func<T, bool>> predicate = null);
       IQueryable<T> GetAllNoEntity(Expression<Func<T, bool>> predicate = null);
       T Get(Expression<Func<T, bool>> predicate);
       Task<T> GetAsyn(Expression<Func<T, bool>> predicate);
       bool IsExist(Expression<Func<T, bool>> predicate);
       IQueryable<T> Get();
       IQueryable<T> GetWithoutShared();
       IQueryable<T> GetNoEntity();
       void Add(T entity);
       void Add(T[] entity);
       void Attach(T entity);
       void Update(T entity);
       void Update(T[] entity);
       void Delete(T entity);
       void Delete(T[] entity);
       void SaveChanges();
       void BulkSaveChanges();
       T Get(int id);
       T Get(long id);
   }
# Repository
  public abstract class Repository<T, PK> : IRepository<T, PK> where T : class
  {
      protected readonly UnitOfWork _unitOfWork;
      readonly protected DbSet<T> _objectSet;
      private readonly IUserInfo _user;


      public Repository(IUnitOfWork unitOfWork)
      {
          _unitOfWork = (UnitOfWork)unitOfWork;
          _user = _unitOfWork.UserInfo;
          _objectSet = unitOfWork.Context._dbContext.Set<T>();
      }
      public IQueryable<T> GetAll(Expression<Func<T, bool>> predicate = null)
      {
          if (predicate != null)
          {
              return this._unitOfWork.Context.Get<T>().Where(predicate).AsQueryable();
          }

          return this._unitOfWork.Context.Get<T>().AsQueryable();
      }

      public IQueryable<T> GetAllNoEntity(Expression<Func<T, bool>> predicate = null)
      {
          if (predicate != null)
          {
              return this._unitOfWork.Context.GetNoEntity<T>().Where(predicate).AsQueryable();
          }

          return this._unitOfWork.Context.GetNoEntity<T>().AsQueryable();
      }

      public T Get(Expression<Func<T, bool>> predicate)
      {
          return this._unitOfWork.Context.Get<T>().FirstOrDefault(predicate);
      }
      public async Task<T> GetAsyn(Expression<Func<T, bool>> predicate)
      {
          return await this._unitOfWork.Context.Get<T>().FirstOrDefaultAsync(predicate);
      }
      public IQueryable<T> Get()
      {
          // var tran = false;
          try
          {
              // tran = this.BeginTran();
              var query = _objectSet;
              if (typeof(T).GetProperty("EntityId") == null)
              {
                  return query;
              }
              else
              {
                  var param = Expression.Parameter(typeof(T), typeof(T).Name);
                  var column = Expression.Property(param, "EntityId");
                  var value = Expression.Convert(Expression.Constant(_user.OrgId), column.Type);
                  var where = Expression.Lambda<Func<T, bool>>(Expression.Equal(column, value), param);
                  return query.Where<T>(where);
              }
          }
          catch (Exception ex)
          {
              //if (tran)
              //{
              //    this.Rollback();
              //}
              throw ex;
          }
          finally
          {
              //if (tran)
              //{
              //    this.Commit();
              //}
          }



          // return this._unitOfWork.Context..Get<T>().AsQueryable();
      }

      public bool IsExist(Expression<Func<T, bool>> predicate = null)
      {
          return _objectSet.Any(predicate);
      }
      public IQueryable<T> GetWithoutShared()
      {
          return this._unitOfWork.Context.GetWithoutShared<T>().AsQueryable();
      }
      public IQueryable<T> GetNoEntity()
      {
          return this._unitOfWork.Context.GetNoEntity<T>().AsQueryable(); ;
      }

      public void Add(T entity)
      {
          if (entity.GetType().GetProperty("EntityId") != null)
          {
              entity.GetType().GetProperty("EntityId").SetValue(entity, _user.OrgId);
          }
          if (entity.GetType().GetProperty("CreatedBy") != null)
          {
              entity.GetType().GetProperty("CreatedBy").SetValue(entity, _user.UserId);
          }
          if (entity.GetType().GetProperty("CreatedDate") != null)
          {
              entity.GetType().GetProperty("CreatedDate").SetValue(entity, DateTime.Now);
          }
          if (entity.GetType().GetProperty("CreatedByEmpId") != null)
          {
              entity.GetType().GetProperty("CreatedByEmpId").SetValue(entity, _user.EmpId);
          }

          _objectSet.Add(entity);
      }

      public void Add(T[] entity)
      {

          _objectSet.AddRange(entity.ToArray());
      }


      public void Attach(T entity)
      {
          _objectSet.Attach(entity);
      }

      public void Update(T entity)
      {
          if (entity.GetType().GetProperty("ConcurrencyStamp") != null)
          {
              entity.GetType().GetProperty("ConcurrencyStamp").SetValue(entity,Guid.NewGuid().ToString());
          }
          _objectSet.Attach(entity);
      }

      public void Update(T[] entity)
      {
          if (entity.GetType().GetProperty("ConcurrencyStamp") != null)
          {
              entity.GetType().GetProperty("ConcurrencyStamp").SetValue(entity, Guid.NewGuid().ToString());
          }
          _objectSet.AttachRange(entity.ToArray());
      }

      public void Delete(T entity)
      {
          _objectSet.Remove(entity);
      }

      public void Delete(T[] entity)
      {
          _objectSet.RemoveRange(entity);
      }


      public void SaveChanges()
      {
          // SetAdministritiveData();
          this._unitOfWork.SaveChanges();
      }

      public void BulkSaveChanges()
      {
          this._unitOfWork.BulkSaveChanges();
      }

      public T Get(int id)
      {
          var query = _objectSet.Find(id);

          if (query != null)
              return query;
          else
              return null;
      }

      public T Get(long id)
      {
          var query = _objectSet.Find(id);

          if (query != null)
              return query;
          else
              return null;
      }


      public void SetAdministritiveData()
      {
          try
          {
              foreach (EntityEntry NewEntry in
                        _unitOfWork.Context._dbContext.ChangeTracker.Entries().Where(a => a.State == EntityState.Added))
              {

                  var EntityID = NewEntry.Entity.GetType().GetProperty("EntityId", BindingFlags.Public | BindingFlags.Instance);
                  if (EntityID != null && EntityID.CanWrite && EntityID.GetValue(NewEntry.Entity) == null)
                  {
                      EntityID.SetValue(NewEntry.Entity, _user.OrgId, null);
                  }
                  var CreatedBy = NewEntry.Entity.GetType().GetProperty("CreatedBy", BindingFlags.Public | BindingFlags.Instance);
                  if (CreatedBy != null && CreatedBy.CanWrite)
                  {
                      CreatedBy.SetValue(NewEntry.Entity, _user.UserId, null);
                  }
                  //rolback
                  var prop = NewEntry.Entity.GetType().GetProperty("CreatedDate", BindingFlags.Public | BindingFlags.Instance);
                  if (prop != null && prop.CanWrite)
                  {
                      prop.SetValue(NewEntry.Entity, DateTime.Now, null);
                  }
                  var IsDeleted = NewEntry.Entity.GetType().GetProperty("IsDeleted", BindingFlags.Public | BindingFlags.Instance);
                  if (IsDeleted != null && IsDeleted.CanWrite)
                  {
                      IsDeleted.SetValue(NewEntry.Entity, false, null);
                  }
                  var CreatedByEmpId = NewEntry.Entity.GetType().GetProperty("CreatedByEmpId", BindingFlags.Public | BindingFlags.Instance);
                  if (CreatedByEmpId != null && CreatedByEmpId.CanWrite)
                  {
                      CreatedByEmpId.SetValue(NewEntry.Entity, _user.EmpId, null);
                  }
                  //Fix Replace Empty String 11-05-2016 Moghazy
                  ReplaceEmptyString(NewEntry);
              }

              foreach (EntityEntry NewEntry in
                         _unitOfWork.Context._dbContext.ChangeTracker.Entries().Where(a => a.State == EntityState.Modified))
              {
                  // to handle concurrency on update 
                  var TimeStamp = NewEntry.Entity.GetType().GetProperty("ModifiedTS", BindingFlags.Public | BindingFlags.Instance);
                  if (TimeStamp != null)
                  {
                      var oldv = NewEntry.OriginalValues.GetValue<byte[]>("ModifiedTS");
                      var newv = NewEntry.CurrentValues.GetValue<byte[]>("ModifiedTS");
                      if (!newv.SequenceEqual(oldv))
                          throw new DBConcurrencyException();
                  }
                  // No need To Edit Entity ID For Edit Action 
                  //var EntityID = NewEntry.Entity.GetType().GetProperty("EntityID", BindingFlags.Public | BindingFlags.Instance);
                  //if (EntityID != null && EntityID.CanWrite)
                  //    EntityID.SetValue(NewEntry.Entity, _user.OrgId, null);
                  var UpdatedBy = NewEntry.Entity.GetType().GetProperty("UpdatedBy", BindingFlags.Public | BindingFlags.Instance);
                  if (UpdatedBy != null && UpdatedBy.CanWrite)
                      UpdatedBy.SetValue(NewEntry.Entity, _user.UserId, null);
                  var prop = NewEntry.Entity.GetType().GetProperty("UpdatedDate", BindingFlags.Public | BindingFlags.Instance);
                  if (prop != null && prop.CanWrite)
                      prop.SetValue(NewEntry.Entity, DateTime.Now, null);

                  ///// in case deletion logically not physically
                  var IsDeleted = NewEntry.Entity.GetType().GetProperty("IsDeleted", BindingFlags.Public | BindingFlags.Instance);
                  if (IsDeleted != null && IsDeleted.CanWrite)
                  {
                      bool oldv, newv;

                      if (NewEntry.OriginalValues["IsDeleted"].GetType() == typeof(bool))
                      {
                          oldv = NewEntry.OriginalValues.GetValue<bool>("IsDeleted");
                          newv = NewEntry.CurrentValues.GetValue<bool>("IsDeleted");
                      }
                      else
                      {
                          oldv = Convert.ToBoolean(NewEntry.OriginalValues.GetValue<bool?>("IsDeleted"));
                          newv = Convert.ToBoolean(NewEntry.CurrentValues.GetValue<bool?>("IsDeleted"));
                      }


                      if (oldv == false && newv == true)
                      {
                          var deletedBy = NewEntry.Entity.GetType().GetProperty("DeletedBy", BindingFlags.Public | BindingFlags.Instance);
                          var deletedDate = NewEntry.Entity.GetType().GetProperty("DeletedDate", BindingFlags.Public | BindingFlags.Instance);
                          if (deletedBy != null && deletedBy.CanWrite)
                          {
                              deletedBy.SetValue(NewEntry.Entity, _user.UserId, null);
                          }
                          if (deletedDate != null && deletedDate.CanWrite)
                          {
                              deletedDate.SetValue(NewEntry.Entity, DateTime.Now, null);
                          }
                      }
                  }
                  //Fix Replace Empty String 11-05-2016 Moghazy
                  //ReplaceEmptyString(NewEntry);
              }
          }
          catch (Exception ex)
          {

          }
      }

      private static void ReplaceEmptyString(EntityEntry entity)
      {
          string str = typeof(string).Name;
          var properties = from p in entity.Entity.GetType().GetProperties()
                           where p.PropertyType.Name == str
                           select p;

          foreach (var item in properties)
          {
              string value = (string)item.GetValue(entity.Entity, null);
              if (value != null && value.Trim().Length == 0)
              {

                  item.SetValue(entity.Entity, null, null);

              }
          }
      }


  }

# IDBRepository
  public interface IDBRepository<T, TKey> : IRepository<T, TKey> where T : class
{
    //IQueryable<T> GetAll(Expression<Func<T, bool>> predicate = null);
    //IQueryable<T> Get();
    //IQueryable<T> GetNoEntity();
    // T Get(Expression<Func<T, bool>> predicate);

    //void Add(T entity);
    //void Delete(T entity);
    //void Update(T entity);
    //void SaveChanges();
}
   # IQueryRepository

    public interface IQueryRepository<T, TKey> : IDBRepository<T, TKey> where T : class
 { 
 }
 # ICommandRepository
 public  interface ICommandRepository<T, TKey> : IDBRepository<T, TKey> where T : class
    {
        void Add(T entity);

        void Attach(T entity);

        void Delete(T entity);

        void SaveChanges();
    }

# EFRepository
     /// <summary>
 /// An implementation for the <code>IGenericRepository</code> interface, it supports the basic CRUD operations
 /// using Entity Framework 6
 /// </summary>
 /// <typeparam name="T">The type of the model class</typeparam>
 /// <typeparam name="TKey">The type of the primary key for the model class</typeparam>
 // ReSharper disable once InconsistentNaming
 public abstract class EFRepository<T, TKey> : ICommandRepository<T, TKey>
     where T : class
 {
     protected readonly IUnitOfWork _unitOfWork;
     readonly protected DbSet<T> _objectSet;
     readonly protected IUserInfo _UserInfo;

     public EFRepository(IUnitOfWork unitOfWork)
     {
         _unitOfWork = unitOfWork;
         _objectSet = unitOfWork.Context.SetDbSet<T>();


     }
     public IQueryable<T> GetAll(Expression<Func<T, bool>> predicate = null)
     {
         if (predicate != null)
         {
             return this._unitOfWork.Context.Get<T>().Where(predicate);
         }

         return this._unitOfWork.Context.Get<T>();
     }

     public IQueryable<T> GetAllNoEntity(Expression<Func<T, bool>> predicate = null)
     {
         if (predicate != null)
         {
             return this._unitOfWork.Context.GetNoEntity<T>().Where(predicate).AsQueryable();
         }

         return this._unitOfWork.Context.GetNoEntity<T>().AsQueryable();
     }

     public T Get(Expression<Func<T, bool>> predicate)
     {
         return this._unitOfWork.Context.Get<T>().FirstOrDefault(predicate);
     }
     public Task<T> GetAsyn(Expression<Func<T, bool>> predicate)
     {
         return this._unitOfWork.Context.Get<T>().FirstOrDefaultAsync(predicate);
     }
     public virtual IQueryable<T> Get()
     {
         return this._unitOfWork.Context.Get<T>().AsQueryable();
     }
     public IQueryable<T> GetWithoutShared()
     {
         return this._unitOfWork.Context.GetWithoutShared<T>().AsQueryable();
     }
     public IQueryable<T> GetNoEntity()
     {
         return this._unitOfWork.Context.GetNoEntity<T>().AsQueryable();
     }

     public virtual void Add(T entity)
     {
         if (entity.GetType().GetProperty("EntityId") != null)
         {
             entity.GetType().GetProperty("EntityId").SetValue(entity, _UserInfo.OrgId);
         }
         if (entity.GetType().GetProperty("CreatedBy") != null)
         {
             entity.GetType().GetProperty("CreatedBy").SetValue(entity, _UserInfo.UserId);
         }
         if (entity.GetType().GetProperty("CreatedDate") != null)
         {
             entity.GetType().GetProperty("CreatedDate").SetValue(entity, DateTime.Now);
         }
         if (entity.GetType().GetProperty("CreatedByEmpId") != null)
         {
             entity.GetType().GetProperty("CreatedByEmpId").SetValue(entity, _UserInfo.EmpId);
         }

         _objectSet.Add(entity);
     }

     public void Attach(T entity)
     {
         _objectSet.Attach(entity);
     }

     public void Delete(T entity)
     {
         _objectSet.Remove(entity);
     }

     public void SaveChanges()
     {
         this._unitOfWork.SaveChanges();
     }

     

     public void Add(T[] entity)
     {
         foreach (var item in entity)
         {


             if (item.GetType().GetProperty("EntityId") != null)
             {
                 item.GetType().GetProperty("EntityId").SetValue(item, _UserInfo.OrgId);
             }
             if (item.GetType().GetProperty("CreatedBy") != null)
             {
                 item.GetType().GetProperty("CreatedBy").SetValue(item, _UserInfo.UserId);
             }
             if (item.GetType().GetProperty("CreatedDate") != null)
             {
                 item.GetType().GetProperty("CreatedDate").SetValue(item, DateTime.Now);
             }

             _objectSet.Add(item);
         }
     }
     public void Delete(T[] entity)
     {
         foreach (var item in entity)
         {
             _objectSet.Remove(item);
         }

     }
     public abstract T Get(int id);
     public abstract T Get(long id);

     public void BulkSaveChanges()
     {
         this._unitOfWork.BulkSaveChanges();
     }

     public bool IsExist(Expression<Func<T, bool>> predicate)
     {
         return _objectSet.Any(predicate);
     }

     public void Update(T[] entity)
     {
         if (entity.GetType().GetProperty("ConcurrencyStamp") != null)
         {
             entity.GetType().GetProperty("ConcurrencyStamp").SetValue(entity, Guid.NewGuid().ToString());
         }
         _objectSet.AttachRange(entity);
     }
     public void Update(T entity)
     {
         if (entity.GetType().GetProperty("ConcurrencyStamp") != null)
         {
             entity.GetType().GetProperty("ConcurrencyStamp").SetValue(entity, Guid.NewGuid().ToString());
         }
         _objectSet.Attach(entity);
     }

     
 }
   

    
    
