sqlalchemy.exc.OperationalError: (sqlite3.OperationalError) no such column: mapping_versions.arq3_import_id
[SQL: SELECT mapping_versions.id AS mapping_versions_id, mapping_versions.tag AS mapping_versions_tag, mapping_versions.science_import_id AS mapping_versions_science_import_id, mapping_versions.portal_import_id AS mapping_versions_portal_import_id, mapping_versions.arq3_import_id AS mapping_versions_arq3_import_id, mapping_versions.mapping_json AS mapping_versions_mapping_json, mapping_versions.join_keys_json AS mapping_versions_join_keys_json, mapping_versions.join_type AS mapping_versions_join_type, mapping_versions.fuzzy_threshold AS mapping_versions_fuzzy_threshold, mapping_versions.rows_science AS mapping_versions_rows_science, mapping_versions.rows_portal AS mapping_versions_rows_portal, mapping_versions.rows_merged AS mapping_versions_rows_merged, mapping_versions.created_at AS mapping_versions_created_at, mapping_versions.is_active AS mapping_versions_is_active 
FROM mapping_versions 
WHERE mapping_versions.is_active = 1 ORDER BY mapping_versions.created_at DESC
 LIMIT ? OFFSET ?]
[parameters: (1, 0)]
(Background on this error at: https://sqlalche.me/e/20/e3q8)

File "C:\Users\40418843\Desktop\novo-modelo-controle-de-rotas-cadastradas\app.py", line 133, in <module>
    elif "Gerar"      in p: render_merge_page()
                            ^^^^^^^^^^^^^^^^^^^
File "C:\Users\40418843\Desktop\novo-modelo-controle-de-rotas-cadastradas\app\ui\pages.py", line 232, in render_merge_page
    merged_df = repo.load_merged_df()
                ^^^^^^^^^^^^^^^^^^^^^
File "C:\Users\40418843\Desktop\novo-modelo-controle-de-rotas-cadastradas\app\db\repository.py", line 287, in load_merged_df
    .order_by(MappingVersion.created_at.desc()).first())
                                                ^^^^^^^
File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\sqlalchemy\orm\query.py", line 2759, in first
    return self.limit(1)._iter().first()  # type: ignore
           ^^^^^^^^^^^^^^^^^^^^^
File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\sqlalchemy\orm\query.py", line 2857, in _iter
    result: Union[ScalarResult[_T], Result[_T]] = self.session.execute(
                                                  ^^^^^^^^^^^^^^^^^^^^^
File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\sqlalchemy\orm\session.py", line 2365, in execute
    return self._execute_internal(
           ^^^^^^^^^^^^^^^^^^^^^^^
File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\sqlalchemy\orm\session.py", line 2251, in _execute_internal
    result: Result[Any] = compile_state_cls.orm_execute_statement(
                          ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\sqlalchemy\orm\context.py", line 306, in orm_execute_statement
    result = conn.execute(
             ^^^^^^^^^^^^^
File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\sqlalchemy\engine\base.py", line 1415, in execute
    return meth(
           ^^^^^
File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\sqlalchemy\sql\elements.py", line 523, in _execute_on_connection
    return connection._execute_clauseelement(
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\sqlalchemy\engine\base.py", line 1637, in _execute_clauseelement
    ret = self._execute_context(
          ^^^^^^^^^^^^^^^^^^^^^^
File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\sqlalchemy\engine\base.py", line 1842, in _execute_context
    return self._exec_single_context(
           ^^^^^^^^^^^^^^^^^^^^^^^^^^
File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\sqlalchemy\engine\base.py", line 1982, in _exec_single_context
    self._handle_dbapi_exception(
File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\sqlalchemy\engine\base.py", line 2351, in _handle_dbapi_exception
    raise sqlalchemy_exception.with_traceback(exc_info[2]) from e
File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\sqlalchemy\engine\base.py", line 1963, in _exec_single_context
    self.dialect.do_execute(
File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\sqlalchemy\engine\default.py", line 943, in do_execute
    cursor.execute(statement, parameters)
