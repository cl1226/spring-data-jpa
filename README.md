spring data jpa 可以通过在接口中按照规定语法创建一个方法进行查询
---
* 接口继承于CrudRepository 或者 PagingAndSortingRepository，JpaRepository,Repository  
  public interface TaskDao extends JpaRepository<Task,Long>{} 

* 或者利用注释的方式表名继承于JpaRepository，例如下面这俩种是等价的 
  @RepositoryDefinition(domainClass = Task.class, idClass = Long.class) 
  public interface TaskDao{}  
  public interface TaskDao extends JpaRepository<Task,Long>{} 
  
 继承CrudRepository 或者 PagingAndSortingRepository，JpaRepository会抽出一些常用的方法，如果你spring data jpa帮你自定义那么多方法，你可以继承于JpaRepository，然后复制一些方法到你的接口中，可以选择性的要一些方法
    @NoRepositoryBean
    interface MyBaseRepository<T, ID extends Serializable> extends Repository<T, ID> {

     T findOne(ID id);

     T save(T entity);
    }
 
    interface TaskDao extends MyBaseRepository<Task, Long> {

    }
 按照规范创建查询方法，一般按照java驼峰式书写规范加一些特定关键字，例如我们想通过任务名来获取任务实体类列表
利用属性获取任务列表
    interface TaskDao extends MyBaseRepository<Task, Long> {
     List<Task> findByName(String name);
    }
 利用and 和 or来获取任务列表
    interface TaskDao extends JpaRepository<Task, Long> {
     List<Task> findByNameAndProjectId(String name,Long projectId);
     List<Task> findByNameOrProjectId(String name,Long projectId);
    }
 利用Pageable ，Sort，Slice获取分页的任务列表和排序
    interface TaskDao extends JpaRepository<Task, Long> {
     Page<Task> findByName(String name,Pageable pageable);
     Slice<Task> findByName(String name, Pageable pageable);
     List<Task> findByName(String name, Sort sort);
    }
 利用Distinct去重
    interface TaskDao extends JpaRepository<Task, Long> {
     List<Person> findDistinctTaskByNameOrProjectId(String name, Long projectId);
    }
 利用OrderBy进行排序
    interface TaskDao extends JpaRepository<Task, Long> {
     List<Person> findByNameOrderByProjectIdDesc(String name, Long projectId);
    }
 利用 Top 和 First来获取限制数据
    interface TaskDao extends JpaRepository<Task, Long> {
      User findFirstByOrderByLastnameAsc();
 
      Task findTopByOrderByNameDesc(String name);

      Page<Task> queryFirst10ByName(String name, Pageable pageable);

      Slice<Task> findTop3ByName(String name, Pageable pageable);

      List<Task> findFirst10ByName(String name, Sort sort);

      List<Task> findTop10ByName(String name, Pageable pageable);
    }
 
那么spring data jpa是怎么通过这些规范来进行组装成查询语句呢？
Spring Data JPA框架在进行方法名解析时，会先把方法名多余的前缀截取掉，比如 find、findBy、read、readBy、get、getBy，然后对剩下部分进行解析。
假如创建如下的查询：findByTaskProjectName()，框架在解析该方法时，首先剔除 findBy，然后对剩下的属性进行解析，假设查询实体为Doc
  1. 先判断 taskProjectName （根据 POJO 规范，首字母变为小写）是否为查询实体的一个属性，如果是，则表示根据该属性进行查询；如果没有该属性，继续第二步；
  2. 从右往左截取第一个大写字母开头的字符串此处为Name），然后检查剩下的字符串是否为查询实体的一个属性，如果是，则表示根据该属性进行查询；如果没有该属性，则重复第二步，继续从右往左截取；最后假设task为查询实体Person的一个属性；
  3. 接着处理剩下部分(ProjectName），先判断 task 所对应的类型是否有projectName属性，如果有，则表示该方法最终是根据 “ Person.task.projectName”的取值进行查询；否则继续按照步骤 2 的规则从右往左截取，最终表示根据 “Person.task.project.name” 的值进行查询。
  4. 可能会存在一种特殊情况，比如 Person包含一个 task 的属性，也有一个 projectName 属性，此时会存在混淆。可以明确在属性之间加上 “_” 以显式表达意图，比如 “findByTask_ProjectName()”
支持的规范表达式，这里以实体为User，有firstName和lastName,age
    表达式             例子                            hql查询语句
    And               findByLastnameAndFirstname…     where x.lastname = ?1 and x.firstname = ?2
    Or                findByLastnameOrFirstname…      where x.lastname = ?1 or x.firstname = ?2
    Is,Equals         findByFirstname,
                      findByFirstnameIs,
                      findByFirstnameEqual…           where x.firstname = 1?
    Between           findByStartDateBetween…         where x.startDate between 1? and ?2
    LessThan          findByAgeLessThan…              where x.age < ?1
    LessThanEqual     findByAgeLessThanEqual…         where x.age ⇐ ?1
    GreaterThan       findByAgeGreaterThan…           where x.age > ?1
    GreaterThanEqual  findByAgeGreaterThanEqual…      where x.age >= ?1
    After             findByStartDateAfter…           where x.startDate > ?1
    Before            findByStartDateBefore…          where x.startDate < ?1
    IsNull            findByAgeIsNull…                where x.age is null
    IsNotNull,NotNull findByAge(Is)NotNull…           where x.age not null
    Like              findByFirstnameLike…            where x.firstname like ?1
    NotLike           findByFirstnameNotLike…         where x.firstname not like ?1
    StartingWith      findByFirstnameStartingWith…    where x.firstname like ?1 (parameter bound with appended %)
    EndingWith        findByFirstnameEndingWith…      where x.firstname like ?1 (parameter bound with prepended %)
    Containing        findByFirstnameContaining…      where x.firstname like ?1 (parameter bound wrapped in %)
    OrderBy           findByAgeOrderByLastnameDesc…   where x.age = ?1 order by x.lastname desc
    Not               findByLastnameNot…              where x.lastname <> ?1
    In                findByAgeIn(Collection ages)…   where x.age in ?1
    NotIn             findByAgeNotIn(Collection age)… where x.age not in ?1
    True              findByActiveTrue()…             where x.active = true
    False             findByActiveFalse()…            where x.active = false
    IgnoreCase        findByFirstnameIgnoreCase…      where UPPER(x.firstame) = UPPER(?1)

