# login_log_sql
```
SELECT ll.id, ll.user_id, ll.macro_program_num, ll.login_date_log, ll.login_status
FROM login_log ll
WHERE ll.login_status = 'login'
AND NOT EXISTS (
    SELECT 1
    FROM login_log logout_check
    WHERE logout_check.user_id = ll.user_id
    AND logout_check.login_date_log > ll.login_date_log
    AND logout_check.login_status = 'logout'
);
```
