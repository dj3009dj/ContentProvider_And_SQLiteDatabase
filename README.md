# ContentProvider-SQLiteDatabase
<h><b>SQLiteDatabase</b><h><br/>
SQLiteDatabase作为Android内置的轻量级数据库，常使用SQLiteOpenHelper建立,SQLiteOpenHelper是个抽象类，需要实现onCreate和onUpdate方法<br/>
<pre>
public class MySQLiteHelper extends SQLiteOpenHelper{
    public static final String TABLE_COMMENTS="commments";
    public static final String COLUMN_ID="_id";//注意通常需要将"_id"命名为主键列
    public static final String COLUMN_COMMENT="comment";
    private static final String DATABASE_NAME="commments.db";
    private static final int DATABASE_VERSION=1;//版本号一般从1开始

    private static final String DATABASE_CREATE="create table "
            +TABLE_COMMENTS+"("+COLUMN_ID+" integer primary key autoincrement, "
            +COLUMN_COMMENT+" text not null);";//SQLiteDatabase只存在integer/text(类似于string)/real(类似于double)三种数据类型
    public MySQLiteHelper(Context context){
      super(context,DATABASE_NAME,null,DATABASE_VERSION);//调用构造器的目的：创建数据库文件
    }
    @Override
    public void onCreate(SQLiteDatabase db) {//当第一次调用SQLiteOpenHelper的getWritableDatabase/getReadableDatabase时，会执行
           db.execSQL(DATABASE_CREATE);//onCreate和onUpgrade方法，onCreate方法用来实现数据库中table的创建。
    }
    @Override
    public void onUpgrade(SQLiteDatabase db, int oldVersion, int newVersion) {
     Log.e(MySQLiteHelper.class.getName(),"Updating database from version "
     +oldVersion+" to "+newVersion+", which will destroy all old data");
        db.execSQL("DROP TABLE IF EXISTS "+TABLE_COMMENTS);//更新数据库文件时，一般先删除表数据，再调用onCreate创建。
        onCreate(db);
    }
}</pre>
<h><b>ContentProvider</b></h><br/>
ContentProvider的最大目的是跨进程调用。首先需要在manifest.xml文件中注册<br/>
<pre>
android:name=".MyProvider"
android:authorities="com.sijunding.provider"//这就是暴露给ContentResolver的Uri
android:multiprocess="false"//如果设为"true",在每个client进程中都会有该provider的实例；而默认的"false"只有包含该provider的应用拥有
android:exported="true"//如果设为"false",其他app不能使用该进程,只有拥有相同UID(user id)的应用可以使用，默认为true
</pre>
插一下Uri的基础知识：<br/>
URI的基本构成：scheme:[//[user:password@]host[:port]][/]path[?query][#fragment]<br/>
scheme就是协议名，比如http；host和port就是主机和端口号，表示要访问的域名；在'/'后面的path，实际上指的是资源部分。e.g：<br/>
<pre>https://www.zhihu.com/question/37885169</pre>使用uri.getPathSegments()得到一个list<string>,其中该list.get(0)就是"question"<br/>
插一下UriMatcher的用法：<br/>
<pre>
private static UriMatcher matcher=new UriMatcher(UriMatcher.NO_MATCH)；
private static final AUTHORITY="com.sijunding.provider";
private static final WORD=10;
static{//静态初始化块
matcher.addURI(AUTHORITY,"word/#",WORD)//最终的URI就是content://com.sijunding.provider/word/#，#是通配符.content是默认伪协议
}//这样当使用matcher.match(Uri.parse("content://com.sijunding.provider/word/#"))时，返回的就是WORD
</pre>
插一下ContentUris的用法：<br/>
Content Uri的常见形式是content://authority/path/id，所以ContentUris.withAppendedId（uri,id)就是将id追加到path的末尾，返回一个新的uri
<br/>
ContentUris.parseId(uri)就是将最后的path segmentf返回成long类型的id<br/>
插一下contentprovider和contentresolver的对应关系:<br/>
<pre>
某个app的provider:	public Cursor query(Uri uri, String[] projection, String where,String[] whereArgs, String sortOrder)
另个app的resolver:  public  Cursor query(Uri uri, String[] projection, String selection, String[] selectionArgs, String sortOrder) 
当resolver传入的uri是content://com.sijunding.provider/word/#时，就可以调用到另外一个app的provider。在provider里面操作SQLiteDatabase
</pre>
解释各样参数如下：<br/>
String[]projection: 返回符合检索条件的Cursor时，每条记录包含的columns，如果为nul代表所有columns<br/>
String[] seletion: 过滤rows(记录)的条件，类似于SQL中的where子句,形如"_id=?"之类的string<br/>
String[] selectionArgs:对应selection中的"?"对应的字符串数组，形如new String[]{"xxx"}<br/>
String sortOrder:符合检索条件的rows在cursor中的排序设定<br/>
得到cursor后将其转换成rows的方式：<br/>
<pre>
while(cursor.moveToNext()){
cursor.getXXX(int columnIndex);//columnIndex就是字段的列索引，从1开始
}</pre>
###############################update####################################<br/>
<pre>
public  Cursor query(Uri uri, String[] projection, String selection, String[] selectionArgs, String sortOrder) 
{
 ...利用UriMatcher.match(uri)判断，调用对应的数据表...
  return database.query(TABLE_COMMENTS,projection,selection,null,null,sortOrder);//groupby和having都设置为null
}</pre>
###############u#######insert###########################################<br/>
<pre>
ContentValues values=new ContentValues();
value.put(COLUMN_ID,10);//values的第一个参数都是String类型的
value.put(COLIMN_COMMENT,"test");//每次只能插成一个row，返回该row的Uri
public Uri insert(Uri uri,ContentValues values){
//SQLiteDatabase.insert的第二个参数是String类型的，由于SQL不允许插入一个null的记录，当values是空的时候
//第二个参数nullColumnHack会被显式插入NULL
long rowId=database.insert(TABLE_COMMENTS,COMMENT_ID,values);
//获取新row的uri
Uri newUri=ContentUris.withAppendedId(uri,rowId);
//插入数据后，需要提醒ContentResolver有一条记录更新了
getContext().getContentResolver().notifyChange(uri,null);
}</pre>
##############这样resolver端使用如下代码insert记录即可#######<br/>
<pre>
//在另个app的Activity中使用的，主要与provider的区别是要加上"content://"伪协议，这样就相当于调用上面的insert方法
this.getContentResolver().insert("content://com.sijunding.provider/word",values);
</pre>
################################delete####################################<br/>
<pre>
public int delete (Uri uri, String selection, String[] selectionArgs)
{
//一般从resolver端传来的uri带有rowId，形如content://contacts/people/22
   long id=ContentUris.parseId(uri);
   selection=selection+" and _id="+id; //就是说在filter出再加上_id的判断
   database.delete(TABLE_COMMENTS,selection,selectionArgs);
   //删除数据后，需要提醒ContentResolver有一条记录更新了
   getContext().getContentResolver().notifyChange(uri,null);
}
</pre>
##############################update#####################################<br/>
<pre>
public int update (Uri uri, ContentValues values, String selection, String[] selectionArgs)
{
//一般从resolver端传来的uri带有rowId，形如content://contacts/people/22
   long id=ContentUris.parseId(uri);
   selection=selection+" and _id="+id; //就是说在filter出再加上_id的判断
   database.update(TABLE_COMMENTS,selection,selectionArgs);
   //更新数据后，需要提醒ContentResolver有一条记录更新了
   getContext().getContentResolver().notifyChange(uri,null);
}
</pre>
