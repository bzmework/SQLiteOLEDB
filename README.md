![SQLiteOLEDB](logo.png)

SQLite OLEDB Provider
=====================

This is a Microsoft Visual C++ 2010 SP1 project for an Updateable SQLite OLEDB Provider that supports most OLEDB data types, Unicode strings and Blobs. To compile you will need to do some minor hacking in oledb.h provided by Microsoft (described further below).

* The project contains a pre-built DLL that you can download and use straight away.

* I have also added a Visual Basic 6.0 demo project but you will need some ActiveX Grid Controls (dxDBGrid) in order to run them.

Provider Objective
------------------
The SQLite OLE DB Provider features I tried to develop are:

- Several Schema Rowsers, in particular Tables, Columns and Foreign Keys for retrieving Table Relations and Lookups.
- Scrollable and updateable Rowsets.
- Support for both Client and Server cursors.
- Support for Batch Optimistic Updates.
- Support for Old Values.
- UTF8 to Unicode and vice/versa support.
- Compatibility with Microsoft Client Cursor Engine for disconnected (client) recordset updates.
- Automatic SELECT @@IDENTITY after inserts.
- Support for BLOBs and variable length strings and binary data.
- Support for all numeric types: DBTYPE_I1, DBTYPE_I2, DBTYPE_I4, DBTYPE_I8, DBTYPE_UI1, DBTYPE_UI2, DBTYPE_UI4, DBTYPE_UI8 and DBTYPE_R4, DBTYPE_R8.
- Support for special types: DBTYPE_CY (Currency) and DBTYPE_DECIMAL (Decimal).
- Support for date-time types: DBTYPE_DBDATE, DBTYPE_DBTIME, DBTYPE_DBTIMESTAMP.
- Support for DBTYPE_BOOL.
- Automatic identity assignment for Autoincrement Identity Columns with Server-side Cursor Rowset.
- Allow multiple inserts on Client-side Rowsets, even with Autoincrment Identity Column.
- Seamless Data Bindings with most old and new high-level programming language controls.
- Stand-alone zip installation and distribution.

Disclaimer
----------

The compiled SQLite OLEDB Provider DLL works with the Sample Project and Demo Database included in this repository, but I haven't tested it with commercial application development and databases. 

I am sure there will be several bugs in the code. My primary objective was to get the architecture right and implement most of the key features so I didn't have time to do excessive testing and debugging.

Sample Application
------------------

![SQLiteOLEDB](SampleApp.jpeg)

The Visual Basic 6.0 SP1 demo project requires the following Commercial ActiveX controls (please obtain a license or run the .exe provided):

* Codejock.Controls.v15.3.1.ocx
* Codejock.PropertyGrid.v15.3.1.ocx
* HexEdit.ocx
* DXDBGrid.dll

Background
----------

I knew nothing about OLE DB Providers so I did some reasearch and hacked some samples I found on the internet. If you are newbe to OLEDB Providers programming like myself, the important bit of information you need to get started is this:

An OLEDB Provider is a COM CoClass (CSQLiteDataSource) that is instantiated when you create an ADODB.Coonection with the following connection string:

    Dim Conn As ADODB.Connection
    Set Conn = New ADODB.Connection
    Conn.ConnectionString = "Provider=SQLiteOLEDBProvider.SQLiteOLEDB.1;Data Source=D:\mobileFX\Projects\Software\SQLiteOLEDB\VBTest\team.db"

The provider creates a session (CSQLiteSession) when you call Connection.Open()

    Conn.Open

Getting the Schema
-------------------------

In most programming cases you will need to query the database and get its schema.

	Set rs = Conn.OpenSchema(adSchemaColumns)

When you try to get Shema Information the Session object needs to return some "Special" recordsets that we need to implement according to ATL templates available:

		SCHEMA_ENTRY(DBSCHEMA_TABLES, CRSchema_DBSCHEMA_TABLES)					// Database Tables
		SCHEMA_ENTRY(DBSCHEMA_COLUMNS, CRSchema_DBSCHEMA_COLUMNS)				// Database Table Columns
		SCHEMA_ENTRY(DBSCHEMA_PROVIDER_TYPES, CRSchema_DBSCHEMA_PROVIDER_TYPES)	// Supported Datatypes

You will find that a particularly nessesary "Special" schema recordset has no template, but I created one for you:

		SCHEMA_ENTRY(DBSCHEMA_FOREIGN_KEYS, CRSchema_DBSCHEMA_FOREIGN_KEYS)		

All the above schema recordsets are offered by the CSQLiteSession class. The ATL template has a bug that fails to run without crashing. You need edit "atldb.h" and either delete or comment out "ATLASSUME(rgRestrictions != NULL);" I have no idea what those restrictions are, I just copy-pasted some code that makes sence in "CheckRestrictions" and "SetRestrictions" and it works.

		#define SCHEMA_ENTRY(guid, rowsetClass) \
			if (ppGuid != NULL && SUCCEEDED(hr)) \
			{ \
				cGuids++; \
				*ppGuid = ATL::AtlSafeRealloc<GUID, ATL::CComAllocator>(*ppGuid, cGuids); \
				hr = (*ppGuid == NULL) ? E_OUTOFMEMORY : S_OK; \
				if (SUCCEEDED(hr)) \
					(*ppGuid)[cGuids - 1] = guid; \
				else \
					return hr; \
			} \
			else \
			{ \
				if (InlineIsEqualGUID(guid, rguidSchema)) \
				{ \
					/*ATLASSUME(rgRestrictions != NULL); */ \	  <----------------------(DELETE)----------------------------
					rowsetClass* pRowset = NULL; \
					hr = CheckRestrictions(rguidSchema, cRestrictions, rgRestrictions); \
					if (FAILED(hr)) \
						return E_INVALIDARG; \
					hr =  CreateSchemaRowset(pUnkOuter, cRestrictions, \
									   rgRestrictions, riid, cPropertySets, \
									   rgPropertySets, ppRowset, pRowset); \
					return hr; \
				} \
			}

As you can imagine by now the CSQLiteSession class does most of the work. The pattern is this:

1. When the session is created by CSQLiteDataSource::CreateSession() we open the SQLite database and keep it open.

2. We need a RecordClass that describes a single Record for every table we access in order to store field information. Luckyly for schema classes there are templates that do that. For real-data classes we need a storage class.

3. We need a RecordSetClass that holds the records. This class is fetching data from SQLite Database and has the following pattern:

		class CRecordSetClassXXX : public CRowsetImpl<CRecordXXX, CTABLESRow, CSQLiteSession>
		{
		public:
		
			LONG record_count;
			CAtlArray<CRecordXXX,CElementTraits<CRecordXXX>>* m_Data;
		
			/////////////////////////////////////////////////////////////////////////////
			DBSTATUS GetDBStatus(CSimpleRow*, ATLCOLUMNINFO* pInfo)
			{		
				return DBSTATUS_S_OK;
			}
		
			/////////////////////////////////////////////////////////////////////////////
			HRESULT Execute(LONG* pcRowsAffected, ULONG, const VARIANT* params)
			{		
				// Get the Session Object
				HRESULT hr;
				CComPtr<IGetDataSource> ipGDS;
				if (FAILED(hr = GetSpecification(IID_IGetDataSource, (IUnknown **)&ipGDS))) return hr;		
				CSQLiteSession *pSess = static_cast<CSQLiteSession *>((IGetDataSource*)ipGDS);
		
				// Execute the SQL
				char *zErrMsg = 0;	
				record_count=0;		
				m_Data = &m_rgRowData;
				int rc = sqlite3_exec(pSess->db, SQL, sqlite_callback, this, &zErrMsg);
				if(rc) return E_FAIL;
				*pcRowsAffected = record_count;
				return S_OK;
			}
		
			//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////	
			static int sqlite_callback(void *NotUsed, int argc, char **argv, char **azColName)	
			{
				CRecordSetClassXXX* objPtr = (CRecordSetClassXXX*) NotUsed;
		
				// Create new record of special data		
				CRecordXXX trData;
		
				// Save value		
				trData.xxxxx = argv[0];						
		
				// Add new Record to storage
				objPtr->m_Data->Add(trData);			
				objPtr->record_count++;
				return 0;
			}
		};

You will notice that data retreival from SQLite Database is done with static callback "sqlite_callback" and we use the 1st parameter of the callback to pass the recordset object instance. Actually I recently changed the SQLite data retreival/update method to a more efficient implementation without static callbacks and ny using compiled statements instead.

Most of recordset data are strings and must be stored as BSTR. Recall to SysAllocFree() your BSTR or you will suffer memory leaks:

		BSTR name = UTF8_to_BSTR(argv[1]);		
		::wcscpy(trData.m_szColumnName, name);
		SysFreeString(name);		

The more "Special" schema Recordsets you implement the more close you get to being compatible with ADO Recordsets and the more your client-code will need less changes, if any to work. However there is a tricky part with ADO Properties but I haven't done anything about them yet.

SELECT-ing data
---------------

Rowsets are the internal structure of ADO Recordsets. In fact, ADO Recordsets are just wrappers that callback the functions of numerous interfaces implemented by a Rowset. A Rowset has many internal banks, often MFC Arrays that maintain operational and functional data such as Rows, Bookmarks and Binders.

A Row is two things: A "Data Row" (also called User Row) that represents a Record and holds the actual data obtained by executing the command's SQL query, and a "Handles Row" that holds the status and the cursor of the Row (aka. Row Index). Thus for every Row in our Rowset we need two (2) objects: 

- A CSQLiteRowsetRowHandle object that holds Row Handle, Status and other control variables.

- A CSQLiteRowsetRowData object that holds that actual data for this Row.

Handle Rows are stored in m_rgRowHandles[] and Data Rows are stored in m_rgRowData[]. Other internal storages are m_rgBookmarks[] that holds the bookmarks and m_rgBindings[] that holds consumer bindings. All those banks are defined in base classes in atldb.h.

VERY IMPORTANT
==============

The objects that compose a Rowset and you need to implement are:

	CSQLiteRowset				- The actual Rowset implantation
	CSQLiteRowsetDataStorage	- The Storage of our Rowset holding all the Rows (Records)
	CSQLiteRowsetRowHandle		- A Data Row of our Rowset
	CSQLiteRowsetRowData		- A Handles Row of our Rowset
	CSQLiteColumnsRowset		- A special Metadata Rowset that holds information about columns
	CSQLiteColumnsRowsetRow		- The Row (record) of our special Metadata Rowset.

CSQLiteRowset opens the SQLite database and reads data. It implements the minimum interfaces for fetching and updating data but most of the work is done by the ATL Templates in oledb.h and the code we need to implement is very straight forward and it has to do with SQLite access and the Columns Recordset.

CSQLiteRowsetRowData which represents a single record holds the data of the columns in a  std::vector<string> and exposes two important functions: read() and write() that perform writting and readding to the consumer buffer (a void* buffer passed from the consumer to our rowset for exchanging data). Both functions perform data conversion from the consumer format to the storage format (std::string).

In general, the consumer (eg. ADO Recordset) reads and writes data to and from a Data Row, and in particular to one column at a time. This means that GetData() and SetData() methods both need to know the Row Index and Column Index in order to access the internal arrays.

	GetData(HROW hRow, HACCESSOR hAccessor, void *pData)
	SetData(HROW hRow, HACCESSOR hAccessor, void *pData)

As you can see both GetData and SetData pass a Row Handle (hRow), an Accessor (hAccessor) and a buffer pointer that is used for reading / writing data. With the passed hRow we lookup m_rgRowHandles[] and retrieve the CSQLiteRowsetRowHandle object that holds the m_iRowset cursor which is the RowIndex. With the passed hAccessor we lookup the m_rgBindings[] and retrieve hAccessor's bindings collection. Normally it contains 1 binding where pBindCur->iOrdinal-1 is the ColIndex but since we might have bookmarks and many bindings in hAccessor we need to loop and also care for bookmarks.

You will need to hack oledb.h: find DB_E_NULLACCESSORNOTSUPPORTED and comment out the assertion to get writable Rowsets that support inserting new records.

In both GetData and SetData void *pData is the consumer buffer; that is where we need to copy or read our records data. Data are defined by 1) a status, 2) their length and 3) their value, so when reading data we need to write to that buffer 3 times and when saving data we need to read from that buffer 3 times. Status is simply used for handling NULL values where length and value should be obvious by now.

An important aspect for reading/writing data is transforming them to and from one datatype to another. For example, I am using std::string vector for keeping all data so if a column is declared as integer I need to properly transcode the string to an integer. Doing this by hand is not good enough, you will eventually have to use IDataConvert::DataConvert() function.	

```
#pragma once

#include <sstream>
#include "SQLiteOLEDB.h"
#include "SQLiteRowsetRow.hpp"

class CSQLiteRowset;

/////////////////////////////////////////////////////////////////////////////
class CSQLiteRowsetRowHandle : public CSimpleRow
{
public:
	ULONG m_Bookmark;

	CSQLiteRowsetRowHandle(ULONG RowIndex) : CSimpleRow(RowIndex)
	{
		m_Bookmark=0;
	}
};

/////////////////////////////////////////////////////////////////////////////
class CSQLiteRowsetRowData : public SQLiteRowsetRow<CSQLiteRowset>
{
public:
};

/////////////////////////////////////////////////////////////////////////////
class CSQLiteRowsetDataStorage : public std::vector<CSQLiteRowsetRowData>	  //CAtlArray<CSQLiteRowsetRowData>
{
public:
	
	CSQLiteRowsetDataStorage()
	{

	}

	~CSQLiteRowsetDataStorage()
	{
		clear();
	}
	
	ULONG GetCount()
	{
		return size();
	}

	void Add(CSQLiteRowsetRowData tr)
	{
		push_back(tr);
	}

	void RemoveAt(ULONG index)
	{
		erase(begin() + index);
	}

	void RemoveAll()
	{
		clear();
	}
};

// ==================================================================================================================================
//	   ___________ ____    __    _ __       ____                          __ 
//	  / ____/ ___// __ \  / /   (_) /____  / __ \____ _      __________  / /_
//	 / /    \__ \/ / / / / /   / / __/ _ \/ /_/ / __ \ | /| / / ___/ _ \/ __/
//	/ /___ ___/ / /_/ / / /___/ / /_/  __/ _, _/ /_/ / |/ |/ (__  )  __/ /_  
//	\____//____/\___\_\/_____/_/\__/\___/_/ |_|\____/|__/|__/____/\___/\__/  
//	                                                                         
// ==================================================================================================================================

/////////////////////////////////////////////////////////////////////////////
class ATL_NO_VTABLE CSQLiteRowset :	
	public CRowsetImpl<CSQLiteRowset, CSQLiteRowsetRowData, CSQLiteCommand, CSQLiteRowsetDataStorage, CSQLiteRowsetRowHandle, IRowsetLocateImpl<CSQLiteRowset, IRowsetExactScroll, CSQLiteRowsetRowHandle>>,

	public IColumnsRowset,
	
	// The OLE DB provider documentation seems to imply that if you want your provider to be updateable 
	// then you need to implement either IRowsetChange or IRowsetUpdate. IRowsetChange has all you need
	// for adding new rows, deleting rows and changing data. IRowsetUpdate adds the ability to batch 
	// together a series of changes and apply them to the data source in one go.

	public IRowsetChangeImpl<CSQLiteRowset, CSQLiteRowsetRowData, IRowsetUpdate, CSQLiteRowsetRowHandle>
{
public:
	
	typedef CRowsetImpl<CSQLiteRowset, CSQLiteRowsetRowData, CSQLiteCommand, CSQLiteRowsetDataStorage, CSQLiteRowsetRowHandle, IRowsetLocateImpl<CSQLiteRowset, IRowsetExactScroll, CSQLiteRowsetRowHandle>> _CSQLiteRowset;
	typedef IRowsetChangeImpl<CSQLiteRowset, CSQLiteRowsetRowData, IRowsetUpdate, CSQLiteRowsetRowHandle> _IRowsetChangeImpl;
	typedef CSQLiteColumnsRowset<CSQLiteCommand> _CSQLiteColumnsRowset;
	typedef CAtlMap<DBCOUNTITEM, CSQLiteRowsetRowHandle*> MapClass;		
	typedef CAtlMap<HACCESSOR, ATLBINDINGS*> BindingVector;				

	/////////////////////////////////////////////////////////////////////////////	
	// Inherited Members
	//
	// m_rgRowHandles
	// m_rgBindings	
	// m_rgRowData	
	// m_rgBookmarks
	//

	/////////////////////////////////////////////////////////////////////////////	
	// Internal Storages 
	sqlite3* db;
	ULONG fieldCount;											// Number of Columns (eq. ADO Fields) in Rowset
	bool m_HasBookmarks;										// Indicates is Rowset uses bookmarks
	
	ATLCOLUMNINFO* m_ColumnsMetadata;
	std::vector<CSQLiteColumnsRowsetRow*> METADATA;

	~CSQLiteRowset()
	{
		//TODO:Delete
		METADATA.clear();
	}
	
	/////////////////////////////////////////////////////////////////////////////	
	BEGIN_COM_MAP(CSQLiteRowset)		
		COM_INTERFACE_ENTRY(IRowsetChange)
		COM_INTERFACE_ENTRY(IRowsetUpdate)
		COM_INTERFACE_ENTRY(IRowsetExactScroll)
		COM_INTERFACE_ENTRY(IRowsetLocate)
		COM_INTERFACE_ENTRY(IColumnsRowset)			
		COM_INTERFACE_ENTRY_CHAIN(_CSQLiteRowset)
	END_COM_MAP()

	ROWSET_PROPSET_MAP(CSQLiteRowset);

// ==================================================================================================================================
//	   _____ ____    __    _ __          ___                           
//	  / ___// __ \  / /   (_) /____     /   | _____________  __________
//	  \__ \/ / / / / /   / / __/ _ \   / /| |/ ___/ ___/ _ \/ ___/ ___/
//	 ___/ / /_/ / / /___/ / /_/  __/  / ___ / /__/ /__/  __(__  |__  ) 
//	/____/\___\_\/_____/_/\__/\___/  /_/  |_\___/\___/\___/____/____/  
//	                                                                   
// ==================================================================================================================================

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////	
// Execute SQLite Query and get data and metadata
//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////	

	HRESULT Execute(DBPARAMS * pParams, LONG* pcRowsAffected)
	{		
		// Execute SQL query on SQLite database and create the OLEDB
		// internal structures required by ATL Templates that represent
		// the columns and rows.

		USES_CONVERSION;

		CComPtr<ICommand> piCommand;
		if (FAILED(GetSpecification(IID_ICommand, (IUnknown **)&piCommand))) return E_FAIL;
		CSQLiteCommand *pCommand = static_cast<CSQLiteCommand *>((ICommand*)piCommand);
		db = pCommand->db;

		// Reset control variables		
		m_NEXT_IDENTITY=0;		
		fieldCount=0;
		m_iRowset = 0;
		m_bCanScrollBack = true;
		m_bCanFetchBack = true;
		m_bRemoveDeleted = true;
		m_bIRowsetUpdate = true;
		m_HasBookmarks = false;
		m_bReset = true;
		m_bIsCommand=1;
		
		int rc = 0;
		sqlite3_stmt* stmt;

		// =========================================================================
		// 1. Check SQL statement
		// =========================================================================
		rc = sqlite3_complete(pCommand->SQL.c_str());
		if(rc==SQLITE_OK)
		{
			// =========================================================================
			// 2. Prepare the SQL statement
			// =========================================================================
			rc = sqlite3_prepare_v2(db, pCommand->SQL.c_str(), -1, &stmt, nullptr);
			if(rc==SQLITE_OK)
			{
				// =========================================================================
				// 3. Get Column Metadata
				// =========================================================================
				DBBYTEOFFSET Offset = 0;
				fieldCount = sqlite3_column_count(stmt);

				// Check if bookmarks are set 			
				CComVariant varBookmarks;
				HRESULT hrLocal = GetPropValue(&DBPROPSET_ROWSET, DBPROP_BOOKMARKS, &varBookmarks);
				m_HasBookmarks = (hrLocal == S_OK &&  varBookmarks.boolVal == VARIANT_TRUE);

				// Allocate memory for Column Metadata
				ULONG szMeta = fieldCount + (m_HasBookmarks?1:0);
				m_ColumnsMetadata = NULL;
				m_ColumnsMetadata = (ATLCOLUMNINFO*)CoTaskMemAlloc(szMeta * sizeof(ATLCOLUMNINFO));				
				memset(m_ColumnsMetadata, 0, szMeta * sizeof(ATLCOLUMNINFO));

				// Need to add bookmark column?				
				if(m_HasBookmarks)
				{
					CSQLiteColumnsRowsetRow* BOOKMARK_COLUMN = new  CSQLiteColumnsRowsetRow();
					if(!BOOKMARK_COLUMN->InitBookmarkColumnMetadata(m_ColumnsMetadata, Offset))
						return E_FAIL;

					// Add bookmark column info to Metadata
					METADATA.push_back(BOOKMARK_COLUMN);
				}

				// Create Metadata object for each Column:								
				for(ULONG i=0;i<fieldCount;i++)
				{
					int Ordinal = i+1;

					ATLCOLUMNINFO& COLUMN_INFO = m_ColumnsMetadata[i+(m_HasBookmarks?1:0)];
					CSQLiteColumnsRowsetRow* DATA_COLUMN = new CSQLiteColumnsRowsetRow();
					
					// Get Column table and name (UTF8)
					const char* table = sqlite3_column_table_name(stmt,i);
					const char* column = sqlite3_column_origin_name(stmt,i);

					if(table!=NULL && column!=NULL)
					{
						std::string base_table_name(table);
						std::string base_column_name(column);
						
						// Now read complete column metadata
						if(!DATA_COLUMN->InitColumnMetadata(db, base_table_name, base_column_name, Ordinal, COLUMN_INFO, Offset))
							return E_FAIL;
					}
					else
					{
						if(!DATA_COLUMN->InitExpressionColumnMetadata(db, stmt, Ordinal, COLUMN_INFO, Offset))
							return E_FAIL;
					}

					// Add data column info to Metadata
					METADATA.push_back(DATA_COLUMN);
				}

				// =========================================================================
				// 4. Execute SQLite Statement and Retrieve Data one row at a time
				// =========================================================================
				for(rc = sqlite3_step(stmt);rc!=SQLITE_OK && rc!=SQLITE_DONE && rc!=SQLITE_ERROR;rc = sqlite3_step(stmt))
				{					
					CSQLiteRowsetRowData tr;
					if(tr.load(this, stmt))
					{
						// Keep a copy of the original data
						tr.OrigData = tr.Data;

						// Save the record
						m_rgRowData.push_back(tr);
					}
					else
					{
						return E_FAIL;	
					}
				}

				// Done with this statement (TODO: use sqlite3_step)
				rc = sqlite3_finalize(stmt);

				// Re-index all bookmarks
				REINDEX_BOOKMARKS();
			}			
		}
		if(rc)
			return E_FAIL;
		// =========================================================================
		
		// Return the number of records "affected"
		if(pcRowsAffected)
			*pcRowsAffected = RECORDS_COUNT();

		return S_OK;
	}

	/////////////////////////////////////////////////////////////////////////////	
	ULONG SQLiteGetIdentity(std::string UTF8_tableName)
	{
		if(UTF8_tableName.size()==0) return -1;
		char* zErrMsg;
		std::string tname = UTF8_tableName.substr(1, UTF8_tableName.size()-2);
		std::string sql("select seq from sqlite_sequence where name='" + tname + "';");		
		LONG max_id = -1;
		int rc = sqlite3_exec(db, sql.c_str(), sqlite_callback_max_id, &max_id, &zErrMsg);
		if(max_id==-1)
		{
			sql = "SELECT MAX(rowid) FROM " + UTF8_tableName;
			rc = sqlite3_exec(db, sql.c_str(), sqlite_callback_max_id, &max_id, &zErrMsg);
		}		
		return max_id==-1 ? 0 : (ULONG) max_id;
	}
	static int sqlite_callback_max_id(void* NotUsed, int argc, char **argv, char **azColName)
	{					
		LONG* max_id = (LONG*)(NotUsed);
		*max_id = argv[0]==NULL ? 0 : atoi(argv[0]);		
		return 0;
	}
	

// ==================================================================================================================================
//	    __  ___                              ___        __  __     __                    
//	   /  |/  /___ _______________  _____   ( _ )      / / / /__  / /___  ___  __________
//	  / /|_/ / __ `/ ___/ ___/ __ \/ ___/  / __ \/|   / /_/ / _ \/ / __ \/ _ \/ ___/ ___/
//	 / /  / / /_/ / /__/ /  / /_/ (__  )  / /_/  <   / __  /  __/ / /_/ /  __/ /  (__  ) 
//	/_/  /_/\__,_/\___/_/   \____/____/   \____/\/  /_/ /_/\___/_/ .___/\___/_/  /____/  
//	                                                            /_/                      
// ==================================================================================================================================

	/////////////////////////////////////////////////////////////////////////////
	void REINDEX_BOOKMARKS()
	{
		if(m_HasBookmarks)
		{
			ULONG record_count = RECORDS_COUNT();
			m_rgBookmarks.SetCount(record_count+4);
			m_rgBookmarks[0] = m_rgBookmarks[1] = m_rgBookmarks[2] = -1;
			for(ULONG i=0;i<record_count; i++)
			{
				m_rgBookmarks[i+3] = i+1;
			}			
		}
	}

	/////////////////////////////////////////////////////////////////////////////
	ULONG RECORDS_COUNT()
	{
		return m_rgRowData.size();
	}

	/////////////////////////////////////////////////////////////////////////////
	ULONG KEY_COLUMN()
	{
		for(ULONG i=0;i<METADATA.size();i++)
			if(METADATA[i]->m_DBCOLUMN_KEYCOLUMN==VARIANT_TRUE)
				return METADATA[i]->m_DBCOLUMN_NUMBER;
		return -1;
	}

	/////////////////////////////////////////////////////////////////////////////
	std::string BASETABLENAME(ULONG Ordinal)
	{
		if(Ordinal<=0) return NULL;
		for(ULONG i=0;i<METADATA.size();i++)
		{
			if(METADATA[i]->m_DBCOLUMN_NUMBER==Ordinal)
			{				
				BSTR b = METADATA[i]->m_DBCOLUMN_BASETABLENAME;				
				return BSTR_to_UTF8(b);
			}
		}
		return std::string("");
	}

	/////////////////////////////////////////////////////////////////////////////	
	std::string NEXT_IDENTITY()
	{	
		if(m_NEXT_IDENTITY==0)
		{
			m_NEXT_IDENTITY = SQLiteGetIdentity(BASETABLENAME(KEY_COLUMN()));
			if(m_NEXT_IDENTITY==0) 
				return std::string("");
		}		
		std::ostringstream sid;
		sid << ++m_NEXT_IDENTITY;
		return sid.str();
	}
	ULONG m_NEXT_IDENTITY;

	/////////////////////////////////////////////////////////////////////////////
	std::string DEFAULT_VALUE(ULONG Ordinal)
	{
		if(Ordinal<=0) 
			return NULL_DATA_VALUE;

		for(ULONG i=0;i<METADATA.size();i++)
		{
			if(METADATA[i]->m_DBCOLUMN_NUMBER==Ordinal && METADATA[i]->m_DBCOLUMN_HASDEFAULT==VARIANT_TRUE)
			{
				//TODO: Implement Defaults
				return NULL_DATA_VALUE;
			}
		}
		return NULL_DATA_VALUE;
	}

// ==================================================================================================================================
//	   ____  __    __________  ____     ____      __            ____                    
//	  / __ \/ /   / ____/ __ \/ __ )   /  _/___  / /____  _____/ __/___ _________  _____
//	 / / / / /   / __/ / / / / __  |   / // __ \/ __/ _ \/ ___/ /_/ __ `/ ___/ _ \/ ___/
//	/ /_/ / /___/ /___/ /_/ / /_/ /  _/ // / / / /_/  __/ /  / __/ /_/ / /__/  __(__  ) 
//	\____/_____/_____/_____/_____/  /___/_/ /_/\__/\___/_/  /_/  \__,_/\___/\___/____/  
//	                                                                                    
// ==================================================================================================================================

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////	
// IColumnsInfo
//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////	

	// Interface is implemented by CRowsetImpl::CRowsetBaseImpl::IColumnsInfoImpl
	// We need to assist it with this static member.

	static ATLCOLUMNINFO* GetColumnInfo(CSQLiteRowset* pv, ULONG* pNumCols)
	{
		ATLASSERT(pv!=NULL);
		ATLASSERT(pNumCols!=NULL);
		return pv->GetSchemaInfo(pNumCols);
	}

	/////////////////////////////////////////////////////////////////////////////
	ATLCOLUMNINFO* GetSchemaInfo(ULONG * pNumCols)
	{		
		*pNumCols = (fieldCount + (m_HasBookmarks ? 1 : 0));
		return m_ColumnsMetadata;
	}

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////	
// IColumnsRowset
//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////	

	// This is a bit complicated to understand but the implementation is actually straight forward:
	// We need to create a Rowset that describes the Columns of our actual Data Rowset. 
	// This rowset is called "ColumnsRowset" and since it is also a rowset it means that it also has Columns and Rows.
	// - The Columns of ColumnsRowset are the properties defined in CSQLiteColumnsRowsetRow (which are the ADO Field Property names)
	// - The Rows of ColumnRowset are the Fields of our Data Rowset (which are the ADO Field Property values)

	/////////////////////////////////////////////////////////////////////////////
	STDMETHODIMP GetAvailableColumns(DBORDINAL *pcOptColumns, DBID **prgOptColumns)
	{
		if(pcOptColumns) *pcOptColumns = 0;
		if(prgOptColumns) *prgOptColumns = 0;
		if(pcOptColumns==NULL || prgOptColumns==NULL) return E_INVALIDARG;
		if(m_bIsCommand && CheckCommandText(GetUnknown())==DB_E_NOCOMMAND) return DB_E_NOCOMMAND;
		if(SQLITE_OPTIONAL_METADATA_COLUMNS!=0)
		{
			DBORDINAL cCols;
			ATLCOLUMNINFO *pColInfo = CSQLiteColumnsRowsetRow::GetColumnInfo(this, &cCols);
			ATLASSERT(cCols==SQLITE_REQUIRED_METADATA_COLUMNS+SQLITE_OPTIONAL_METADATA_COLUMNS);
			*prgOptColumns = (DBID *)CoTaskMemAlloc(SQLITE_OPTIONAL_METADATA_COLUMNS*sizeof(DBID));
			if(!*prgOptColumns)	return E_OUTOFMEMORY;
			*pcOptColumns = SQLITE_OPTIONAL_METADATA_COLUMNS;
			for(DBORDINAL i=0;i<SQLITE_OPTIONAL_METADATA_COLUMNS;i++)
			{
				memcpy(*prgOptColumns+i, &(pColInfo[SQLITE_REQUIRED_METADATA_COLUMNS+i].columnid), sizeof(DBID));			
			}
		}
		return S_OK;
	}

	/////////////////////////////////////////////////////////////////////////////
	STDMETHODIMP GetColumnsRowset(IUnknown *pUnkOuter, ULONG cOptColumns, const DBID rgOptColumns[], REFIID riid, ULONG cPropertySets, DBPROPSET rgPropertySets[], IUnknown **ppColRowset)	
	{
		HRESULT hr = S_OK, hrProps;

		if(ppColRowset==NULL || (cPropertySets>0 && rgPropertySets==NULL) || (cOptColumns>0 && rgOptColumns==NULL))
			return E_INVALIDARG;

		*ppColRowset = NULL;

		for(ULONG i=0;i<cPropertySets;i++)
		{
			if(rgPropertySets[i].cProperties>0 && rgPropertySets[i].rgProperties==NULL)
				return E_INVALIDARG;

			for (ULONG j=0; j < rgPropertySets[i].cProperties; j++)
			{
				DBPROPOPTIONS option = rgPropertySets[i].rgProperties[j].dwOptions;
				if (option != DBPROPOPTIONS_REQUIRED &&	option != DBPROPOPTIONS_OPTIONAL)
					return DB_E_ERRORSOCCURRED;
			}
		}

		if(pUnkOuter && !InlineIsEqualUnknown(riid))
			return DB_E_NOAGGREGATION;

		// Create the ColumnsRowset
		CComPolyObject<_CSQLiteColumnsRowset>* pPolyObj;
		hr = CComPolyObject<_CSQLiteColumnsRowset>::CreateInstance(pUnkOuter, &pPolyObj);
		if(FAILED(hr)) return hr;

		// Initialize Rowset
		CComPtr<IUnknown> spAutoReleaseUnk;
		hr = pPolyObj->QueryInterface(&spAutoReleaseUnk);
		if(FAILED(hr))
		{
			delete pPolyObj;
			return hr;
		}
		_CSQLiteColumnsRowset* pRowsetObj = &(pPolyObj->m_contained);
		hr = pRowsetObj->FInit();
		if(FAILED(hr)) return hr;

		// Mark this Rowset as PropSetRowset and set Properties
		const GUID *ppGuid[1];
		ppGuid[0] = &DBPROPSET_ROWSET;
		hr = pRowsetObj->SetPropertiesArgChk(cPropertySets, rgPropertySets);
		if(FAILED(hr)) return hr;
		hrProps = pRowsetObj->SetProperties(0, cPropertySets, rgPropertySets, 1, ppGuid, true);
		if(FAILED(hrProps)) return hr;

		// Columns Rowset
		CComPtr<IUnknown> spOuterUnk;
		this->QueryInterface(__uuidof(IUnknown), (void **)&spOuterUnk);
		pRowsetObj->SetSite(spOuterUnk);

		// Check to make sure we set any 'post' properties based on the riid requested.
		if(FAILED(pRowsetObj->OnInterfaceRequested(riid))) return hr;

		// Determine columns to create (optional and non-optional)
		bool bAll = false;
		if(cOptColumns==0)
		{
			bAll = true;
			cOptColumns = SQLITE_OPTIONAL_METADATA_COLUMNS;
		}

		// Populate ColumnsRowset Rows
		for(UINT i=0;i<METADATA.size();i++)
		{
			CSQLiteColumnsRowsetRow crrData;
			crrData  = *(METADATA[i]);
			pRowsetObj->m_rgRowData.Add(crrData);
		}

		// Populate ColumnsRowset Columns
		DBORDINAL tmp;
		pRowsetObj->m_rgColumns = new ATLCOLUMNINFO[SQLITE_REQUIRED_METADATA_COLUMNS+cOptColumns];
		if(!pRowsetObj->m_rgColumns) return E_OUTOFMEMORY;		
		ATLCOLUMNINFO *pInfo = _CSQLiteColumnsRowset::_StorageClass::GetColumnInfo(this, &tmp);
		pRowsetObj->m_cColumns = SQLITE_REQUIRED_METADATA_COLUMNS + cOptColumns;
		memcpy(pRowsetObj->m_rgColumns, pInfo, sizeof(ATLCOLUMNINFO)*SQLITE_REQUIRED_METADATA_COLUMNS);
		for(DBORDINAL i=0;i<cOptColumns;i++)
		{
			DBORDINAL j = SQLITE_REQUIRED_METADATA_COLUMNS;
			if(bAll)
			{
				j += i;
			}
			else
			{
				while(j<SQLITE_REQUIRED_METADATA_COLUMNS+SQLITE_OPTIONAL_METADATA_COLUMNS)
				{
					if(memcmp(&pInfo[j].columnid, rgOptColumns+i, sizeof(DBID))==0)
						break;
					j++;
				}
			}
			if(j==SQLITE_REQUIRED_METADATA_COLUMNS+SQLITE_OPTIONAL_METADATA_COLUMNS)
				return DB_E_BADCOLUMNID;
			memcpy(pRowsetObj->m_rgColumns+SQLITE_REQUIRED_METADATA_COLUMNS+i, pInfo+j, sizeof(ATLCOLUMNINFO));
			pRowsetObj->m_rgColumns[SQLITE_REQUIRED_METADATA_COLUMNS+i].iOrdinal = SQLITE_REQUIRED_METADATA_COLUMNS+i+1;
		}

		// Return the Columns Rowset
		if(InlineIsEqualGUID(riid, IID_NULL)) return E_NOINTERFACE;
		hr = pPolyObj->QueryInterface(riid, (void  **)ppColRowset);
		if(FAILED(hr))
		{
			*ppColRowset = NULL;
			return hr;
		}

		return S_OK;
	}

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////	
// IRowsetExactScroll & IRowsetScroll
//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////	

	/////////////////////////////////////////////////////////////////////////////
	STDMETHOD(GetRowsAtRatio)(HWATCHREGION hReserved1, HCHAPTER hChapter, ULONG ulNumerator, ULONG ulDenominator, LONG cRows, ULONG *pcRowsObtained, HROW **prghRows)
	{
		return E_NOTIMPL;
	}

	/////////////////////////////////////////////////////////////////////////////
	STDMETHOD(GetExactPosition)(HCHAPTER hChapter, ULONG cbBookmark, const BYTE *pBookmark, ULONG *pulPosition, ULONG *pcRows)
	{
		return GetApproximatePosition(hChapter, cbBookmark, pBookmark, pulPosition, pcRows);
	}

	/////////////////////////////////////////////////////////////////////////////
	STDMETHODIMP GetApproximatePosition(HCHAPTER hChapter, ULONG cbBookmark, const BYTE *pBookmark, ULONG *pulPosition, ULONG *pcRows)
	{
		if (cbBookmark == 0)
		{
			if (pulPosition) *pulPosition = 0;
			if (pcRows)      *pcRows = RECORDS_COUNT();
		}
		else 
		{
			if (!pBookmark) return E_INVALIDARG;
			if (pulPosition)
			{
				if (cbBookmark == 1)
				{
					if (*(DBBOOKMARK*)pBookmark == DBBMK_FIRST)
					{
						*pulPosition = 1;
					}
					else if (*(DBBOOKMARK*)pBookmark == DBBMK_LAST)
					{
						*pulPosition = RECORDS_COUNT();
					}
					else if (*(DBBOOKMARK*)pBookmark == DBBMK_INVALID)
					{
						return DB_E_BADBOOKMARK;
					}
				}
				else if (cbBookmark == sizeof(DWORD))
				{
					DWORD bookmark = *(DWORD*)pBookmark;
					if (bookmark < 1 || bookmark > RECORDS_COUNT()) return DB_E_BADBOOKMARK;
					*pulPosition = bookmark; 
				}
				else 
					return E_FAIL;
			}
			if (pcRows)	*pcRows = RECORDS_COUNT();
		}
		return S_OK;
	}

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////	
// IRowset & IRowsetChange
//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////	

	STDMETHODIMP ReadWriteData(HROW hRow, HACCESSOR hAccessor, void *pData, bool Reading, bool OriginalData = false)
	{
		HRESULT hr = S_OK;

		// Get Row Cursor (Row Index)
		CSQLiteRowsetRowHandle* pHRow = NULL;
		m_rgRowHandles.Lookup(hRow, pHRow);
		if(!pHRow) return DB_E_BADROWHANDLE;
		if(pHRow->m_iRowset >= RECORDS_COUNT()) return DB_E_DELETEDROW;
		if(pHRow->m_status==DBPENDINGSTATUS_INVALIDROW || pHRow->m_status==DBPENDINGSTATUS_DELETED) return DB_E_DELETEDROW;
		if(!Reading && pHRow->m_status!=DBPENDINGSTATUS_NEW) pHRow->m_status = DBPENDINGSTATUS_CHANGED;
		ULONG RowIndex = pHRow->m_iRowset;
		DBROWCOUNT dwBookmark = pHRow->m_Bookmark;		

		// Loop to read/write Data on Column
		ATLBINDINGS* pBinding = NULL;
		m_rgBindings.Lookup((int)hAccessor, pBinding);
		if(!pBinding) return DB_E_BADACCESSORHANDLE;
		if(pData==NULL && pBinding->cBindings!=0) return E_INVALIDARG;
		CSQLiteRowsetRowData* pDataRow = &(m_rgRowData[RowIndex]);		
		for(DBCOUNTITEM i=0;i<pBinding->cBindings;i++)
		{
			DBBINDING *pBindCur = &(pBinding->pBindings[i]);			
			if(Reading)
				hr = pDataRow->read(this, pData, pBindCur, dwBookmark, OriginalData);
			else
				hr = pDataRow->write(this, pData, pBindCur, dwBookmark);
			if(hr!=S_OK) break;
		}

		return hr;
	}

	/////////////////////////////////////////////////////////////////////////////
	STDMETHODIMP GetData(HROW hRow, HACCESSOR hAccessor, void *pDstData)
	{
		return ReadWriteData(hRow,hAccessor,pDstData,true);
	}

	/////////////////////////////////////////////////////////////////////////////
	STDMETHODIMP SetData(HROW hRow, HACCESSOR hAccessor, void *pData)
	{	
		return ReadWriteData(hRow,hAccessor,pData,false);
	}

	/////////////////////////////////////////////////////////////////////////////
	STDMETHODIMP InsertRow(HCHAPTER hReserved, HACCESSOR hAccessor, void  *pData, HROW *phRow)
	{		
		ULONG record_count = RECORDS_COUNT();

		// ---------------------------------------------------
		// Create new Data Row
		// ---------------------------------------------------
		CSQLiteRowsetRowData tr;		
		ULONG KF = KEY_COLUMN();
		for(ULONG i=0; i<fieldCount; i++)
		{
			tr.Data.push_back( i==KF-1 && KF>0 ?  NEXT_IDENTITY() : DEFAULT_VALUE(i+1) ); 
		}

		// Keep a copy of the original data
		tr.OrigData = tr.Data;

		// Store the record
		m_rgRowData.push_back(tr);

		// ---------------------------------------------------
		// Create new Handle Row
		// ---------------------------------------------------
		DBCOUNTITEM Rows = 0;
		CSQLiteRowsetRowHandle::KeyType key = record_count+1;
		CSQLiteRowsetRowHandle* pRow = new CSQLiteRowsetRowHandle(record_count);
		pRow->m_status = hAccessor ? DBPENDINGSTATUS_NEW : DBPENDINGSTATUS_UNCHANGED;
		pRow->m_Bookmark = record_count+3;
		m_rgRowHandles.SetAt(key, pRow);		
		pRow->AddRefRow();
		if(hAccessor) *phRow=key;

		// ---------------------------------------------------
		// Re-Index Bookmarks
		// ---------------------------------------------------
		if(m_HasBookmarks)
		{
			if(m_rgBookmarks.GetCount()==0)
			{
				m_rgBookmarks.SetCount(record_count+4);
				m_rgBookmarks[0] = m_rgBookmarks[1] = m_rgBookmarks[2] = -1;
				m_rgBookmarks[3] = 1;
			}
			else
			{
				m_rgBookmarks.SetCount(record_count+4);
				m_rgBookmarks[record_count+3] = record_count+1;
			}			
		}

		return S_OK;
	}

	/////////////////////////////////////////////////////////////////////////////
	STDMETHODIMP DeleteRows(HCHAPTER hReserved, DBCOUNTITEM cRows, const HROW rghRows[], DBROWSTATUS rgRowStatus[])
	{	
		return IRowsetChangeImpl::DeleteRows(hReserved, cRows, rghRows, rgRowStatus);
	}

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////	
// IRowsetUpdate
//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////	
					
	/////////////////////////////////////////////////////////////////////////////
	STDMETHODIMP Update(HCHAPTER hReserved, DBCOUNTITEM cRows, const HROW rghRows[], DBCOUNTITEM *pcRows, HROW **prgRows, DBROWSTATUS **prgRowStatus) 
	{
		//TODO: This currently works for adLockBatchOptimistic only.

		// Get number of rows and allocate memory for return buffers
		DBCOUNTITEM ulRows = (DBCOUNTITEM) m_rgRowHandles.GetCount();
		*prgRows = (HROW*)CoTaskMemAlloc(ulRows * sizeof(HROW));
		*prgRowStatus = (DBROWSTATUS*)CoTaskMemAlloc(ulRows * sizeof(DBROWSTATUS));

		// Loop on all rows and update
		std::stringstream SQL;
		POSITION pos = m_rgRowHandles.GetStartPosition();
		for(ULONG ulRow=0; ulRow<ulRows; ulRow++)
		{			
			MapClass::CPair* pPair = m_rgRowHandles.GetNext(pos);			
			HROW hRowUpdate = pPair->m_key;
			CSQLiteRowsetRowHandle* pRow = NULL;
			bool bFound = m_rgRowHandles.Lookup((ULONG)hRowUpdate, pRow);
			if(!bFound) return E_FAIL;
			(*prgRows)[ulRow] = hRowUpdate;
			CSQLiteRowsetRowData* pDataRow = &(m_rgRowData[pRow->m_iRowset]);						
			(*prgRowStatus)[ulRow] = pRow->m_status = pDataRow->update(this, pRow, SQL);			
		}
		*pcRows = ulRows;
		
		// Convert Unicode string to UTF8		
		std::string sql = SQL.str();
		OutputDebugStringA(sql.c_str());
		char* utf8 = STR_TO_UTF8(sql);
		char* zErrMsg;		
		int rc = sqlite3_exec(db, utf8, NULL, NULL, &zErrMsg);		
		free(utf8);
		return (rc==SQLITE_OK ? S_OK : E_FAIL);
	}

	/////////////////////////////////////////////////////////////////////////////
	STDMETHODIMP Undo(HCHAPTER hReserved, DBCOUNTITEM cRows, const HROW rghRows[], DBCOUNTITEM *pcRowsUndone, HROW **prgRowsUndone, DBROWSTATUS **prgRowStatus) 
	{
		return E_NOTIMPL;
	}

	/////////////////////////////////////////////////////////////////////////////
	STDMETHODIMP GetOriginalData(HROW hRow, HACCESSOR hAccessor, void *pData)
	{		
		return ReadWriteData(hRow,hAccessor,pData,true,true);
	}
	
	/////////////////////////////////////////////////////////////////////////////
	STDMETHODIMP GetPendingRows(HCHAPTER hReserved, DBPENDINGSTATUS dwRowStatus, DBCOUNTITEM *pcPendingRows, HROW **prgPendingRows, DBPENDINGSTATUS **prgPendingStatus)
	{		
		bool bPending = false;
		CSQLiteRowsetRowHandle *pRow = NULL;

		if(pcPendingRows)
		{
			*pcPendingRows = 0;
			if(prgPendingRows) *prgPendingRows = NULL;
			if(prgPendingStatus) *prgPendingStatus = NULL;
		}

		// Validate input parameters
		if ((dwRowStatus & 	~(DBPENDINGSTATUS_NEW | DBPENDINGSTATUS_CHANGED | DBPENDINGSTATUS_DELETED)) != 0)
			return E_INVALIDARG;

		// Determine how many rows we'll need to return
		POSITION pos = m_rgRowHandles.GetStartPosition();
		while( pos != NULL )
		{
			MapClass::CPair* pPair = m_rgRowHandles.GetNext( pos );
			ATLASSERT( pPair != NULL );

			// Check to see if a row has a pending status
			pRow = pPair->m_value;
			if (pRow->m_status & dwRowStatus)
			{
				if (pcPendingRows != NULL)
					(*pcPendingRows)++;
				bPending = true;
			}
		}

		// In this case, there are no pending rows that match, just exit out
		if (!bPending)
		{
			// There are no pending rows so exit immediately
			return S_FALSE;
		}
		else
		{
			// Here' the consumer just wants to see if there are pending rows we know that so we can exit
			if (pcPendingRows == NULL)
				return S_OK;
		}

		// Allocate arrays for pending rows
		{
			if (prgPendingRows != NULL)
			{
				*prgPendingRows = (HROW*)CoTaskMemAlloc(*pcPendingRows * sizeof(HROW));
				if (*prgPendingRows == NULL)
				{
					*pcPendingRows = 0;
					return E_OUTOFMEMORY;
				}
			}

			if (prgPendingStatus != NULL)
			{
				*prgPendingStatus = (DBPENDINGSTATUS*)CoTaskMemAlloc(*pcPendingRows * sizeof(DBPENDINGSTATUS));
				if (*prgPendingStatus == NULL)
				{
					*pcPendingRows = 0;
					CoTaskMemFree(*prgPendingRows);
					*prgPendingRows = NULL;
					return E_OUTOFMEMORY;
				}
				memset(*prgPendingStatus, 0, *pcPendingRows * sizeof(DBPENDINGSTATUS));
			}
		}

		if (prgPendingRows || prgPendingStatus)
		{
			ULONG ulRows = 0;
			pos = m_rgRowHandles.GetStartPosition();
			while( pos != NULL )
			{
				MapClass::CPair* pPair = m_rgRowHandles.GetNext( pos );
				ATLASSERT( pPair != NULL );

				pRow = pPair->m_value;
				if (pRow->m_status & dwRowStatus)
				{
					// Add the output row
					pRow->AddRefRow();
					if (prgPendingRows)
						((*prgPendingRows)[ulRows]) = /*(HROW)*/pPair->m_key;
					if (prgPendingStatus)
						((*prgPendingStatus)[ulRows]) = (DBPENDINGSTATUS)pRow->m_status;
					ulRows++;
				}
			}
			if (pcPendingRows != NULL)
				*pcPendingRows = ulRows;
		}

		// Return code depending on
		return S_OK;		
	}

	/////////////////////////////////////////////////////////////////////////////
	STDMETHODIMP GetRowStatus(HCHAPTER hReserved, DBCOUNTITEM cRows, const HROW rghRows[], DBPENDINGSTATUS rgPendingStatus[]) 
	{
		bool bSucceeded = true;
		ULONG ulFetched = 0;

		if(cRows)
		{
			if(rghRows==NULL || rgPendingStatus==NULL) return E_INVALIDARG;
			for (ULONG ulRows=0; ulRows < cRows; ulRows++)
			{
				CSQLiteRowsetRowHandle* pRow;
				bool bFound = m_rgRowHandles.Lookup((ULONG)rghRows[ulRows], pRow);
				if ((! bFound || pRow == NULL) || (pRow->m_status == DBPENDINGSTATUS_INVALIDROW))
				{
					rgPendingStatus[ulRows] = DBPENDINGSTATUS_INVALIDROW;
					bSucceeded = false;
					continue;
				}
				if (pRow->m_status != 0)
					rgPendingStatus[ulRows] = pRow->m_status;
				else
					rgPendingStatus[ulRows] = DBPENDINGSTATUS_UNCHANGED;

				ulFetched++;
			}
		}

		if (bSucceeded)
		{
			return S_OK;
		}
		else
		{
			if (ulFetched > 0)
				return DB_S_ERRORSOCCURRED;
			else
				return DB_E_ERRORSOCCURRED;
		}
	}
};

```

These options are used to specify the characteristics of the static, keyset, and dynamic cursors defined in ODBC as follows:

Static cursor	
-------------

In a static cursor, the membership, ordering, and values of the rowset is fixed after the rowset is opened. Rows updated, deleted, or inserted after the rowset is opened are not visible to the rowset until the command is re-executed.
				
To obtain a static cursor, the application sets the properties:	

	DBPROP_CANSCROLLBACKWARDS to VARIANT_TRUE
	DBPROP_OTHERINSERT to VARIANT_FALSE
	DBPROP_OTHERUPDATEDELETE to VARIANT_FALSE

In ODBC, this is equivalent to specifying SQL_CURSOR_STATIC for the SQL_ATTR_CURSOR_TYPE attribute in a call to SQLSetStmtAttr.

Keyset-driven cursor	
--------------------

In a keyset-driven cursor, the membership and ordering of rows in the rowset are fixed after the rowset is opened. However, values within the rows can change after the rowset is opened, including the entire row that is being deleted. Updates to a row are visible the next time the row is fetched, but rows inserted after the rowset is opened are not visible to the rowset until the command is reexecuted.
					
To obtain a keyset-driven cursor, the application sets the properties:
					
	DBPROP_CANSCROLLBACKWARDS to VARIANT_TRUE
	DBPROP_OTHERINSERT to VARIANT_FALSE
	DBPROP_OTHERUPDATEDELETE to VARIANT_TRUE
					
In ODBC, this is equivalent to specifying SQL_CURSOR_KEYSET_DRIVEN for the SQL_ATTR_CURSOR_TYPE attribute in a call to SQLSetStmtAttr.

Dynamic cursor
--------------
	
In a dynamic cursor, the membership, ordering, and values of the rowset can change after the rowset is opened. The row updated, deleted, or inserted after the rowset is opened is visible to the rowset the next time the row is fetched.

To obtain a dynamic cursor, the application sets the properties:
					
	DBPROP_CANSCROLLBACKWARDS to VARIANT_TRUE
	DBPROP_OTHERINSERT to VARIANT_TRUE
	DBPROP_OTHERUPDATEDELETE to VARIANT_TRUE

In ODBC, this is equivalent to specifying SQL_CURSOR_DYNAMIC for the SQL_ATTR_CURSOR_TYPE attribute in the call to SQLSetStmtAttr.

Cursor sensitivity
------------------

If the rowset property DBPROP_OWNINSERT is set to VARIANT_TRUE, the rowset can see its own inserts; if the rowset property DBPROP_OWNUPDATEDELETE is set to VARIANT_TRUE, the rowset can see its own updates and deletes. These are equivalent to the presence of the SQL_CASE_SENSITIVITY_ADDITIONS bit and a combination of the SQL_CASE_SENSITIVITY_UPDATES and SQL_CASE_SENSITIVITY_DELETIONS bits that are returned in the ODBC SQL_STATIC_CURSOR_ATTRIBUTES2 SQLGetInfo request.

Must read References
--------------------

http://msdn.microsoft.com/en-us/library/windows/desktop/ms713643(v=vs.85).aspx
http://msdn.microsoft.com/en-us/library/ms811710.aspx
http://devzone.advantagedatabase.com/dz/webhelp/Advantage7.1/mergedProjects/adsoledb/adsoledb/rowset_properties.htm
http://interested.googlecode.com/svn/trunk/SQLCEHelper/Source/DbValue.cpp
http://msdn.microsoft.com/en-us/library/windows/desktop/ms723069(v=vs.85).aspx
http://msdn.microsoft.com/en-us/library/windows/desktop/ms715968(v=vs.85).aspx
http://msdn.microsoft.com/en-us/library/windows/desktop/ms714373(v=vs.85).aspx
http://msdn.microsoft.com/en-us/library/windows/desktop/ms712925(v=vs.85).aspx


Test SQL
---------

		DROP TABLE IF EXISTS TEST;
		
		CREATE TABLE TEST ( 
		    ID               INTEGER           PRIMARY KEY AUTOINCREMENT
		                                       NOT NULL
		                                       UNIQUE,
		    [BOOLEAN]        BOOLEAN,
		    DATE             DATE,
		    DATETIME         DATETIME,
		    TIME             TIME,
		    TIMESTAMP        TIMESTAMP,
		    INT              INT,
		    INT2             INT2,
		    INT4             INT4,
		    INT8             INT8,
		    UINT2            UINT2,
		    UINT4            UINT4,
		    UINT8            UINT8,
		    REAL             REAL,
		    DOUBLE           DOUBLE,
		    FLOAT            FLOAT,
		    [DECIMAL(19,2)]  DECIMAL( 19, 2 ),
		    [NUMERIC(19,2)]  NUMERIC( 19, 2 ),
		    MONEY            MONEY,
		    [MONEY(19,2)]    MONEY( 19, 2 ),
		    TEXT             TEXT,
		    CHAR             CHAR,
		    VARCHAR          VARCHAR,
		    [VARCHAR(2)]     VARCHAR( 2 ),
		    [VARCHAR(250)]   VARCHAR( 250 ),
		    [CHAR(250)]      CHAR( 250 ),
		    [TEXT(250)]      TEXT( 250 ),
		    Ελληνικά         VARCHAR,
		    BLOB             BLOB,
		    [VARBYTES(1024)] VARBYTES( 1024 ),
		    IMAGE            IMAGE 
		);
		
		INSERT INTO TEST ([BOOLEAN],[Ελληνικά]) VALUES(1,'Τραγωδία');


<script async="async" src="https://www.paypalobjects.com/js/external/paypal-button.min.js?merchant=epolitakis@mobilefx.com" 
    data-button="donate" 
    data-name="SQLite OLEDB Provider Project Support Donation" 
    data-quantity="1" 
    data-amount="5" 
    data-currency="EUR"
></script>

Author
-------

Elias Politakis,
mobileFX CTO/Partner,
52 Electras str, Kallithea 17673, Greece.

www.mobilefx.com

Credits
-------
Special thanks to Vladimir Vissoultchev for supporting this project.
