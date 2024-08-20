# AutoMapper

- 作用：将一个对象属性映射到另一个对象属性上。

  ```c#
  public class ProductMapping : Profile
  {
      public ProductMapping()
      {
          CreateMap<Product, ProductDto>().ReverseMap();
      }
  }
  
  [Table("product")]
  public class Product : IEntity
  {
      [Column("id")]
      public Guid Id { get; set; }
      
      [Column("name")]
      public string Name { get; set; }
      
      [Column("price")]
      public decimal Price { get; set; }
  }
  
  public class ProductDto
  {
      public Guid Id { get; set; }
  
      public string Name { get; set; }
  
      public decimal Price { get; set; }
  }
  ```

  