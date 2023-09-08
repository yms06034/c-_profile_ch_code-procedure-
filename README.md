# c-_profile_ch_code-procedure-
```
 /// <summary>
    /// 프로필 이미지 변경
    /// </summary>
    /// <param name="file"></param>
    /// <returns></returns>
    /// <remarks>
    ///
    ///
    ///  extension supported :
    ///
    /// 
    ///         jpg or jpeg or png
    /// </remarks>
    /// <exception cref="ApiException"></exception>
    [HttpPost("profileImg")]
    [ProducesResponseType(StatusCodes.Status200OK, Type = typeof(string))]
    [ProducesResponseType(StatusCodes.Status400BadRequest, Type = typeof(string))]
    [ProducesResponseType(StatusCodes.Status404NotFound, Type = typeof(string))]
    [ProducesResponseType(StatusCodes.Status500InternalServerError)]
    public async Task<IActionResult> SaveProfileImg([Required] IFormFile file)
    {
        if (!Path.GetExtension(file.FileName).Equals(".jpg", StringComparison.OrdinalIgnoreCase) &&
            !Path.GetExtension(file.FileName).Equals(".jpeg", StringComparison.OrdinalIgnoreCase) &&
            !Path.GetExtension(file.FileName).Equals(".png", StringComparison.OrdinalIgnoreCase))
        {
            throw new ApiException((int)HttpStatusCode.UnsupportedMediaType, HttpStatusCode.UnsupportedMediaType,
                GetEnumDesc.GetEnumDescription(ExceptionEnum.EXTENSION_NOT_SUPPORTED));
        }

        var userId = Helper.GetRequestInfo(Request).UserId;

        var saveProfileImgNm = await _userInfoService.saveProfile(userId, file);

        return CustomResult($"{userId} 님의 프로필 이미지가 변경되었습니다. ", saveProfileImgNm, HttpStatusCode.OK);
    }

public async Task<string> saveProfile(string userId, IFormFile file)
    {
        try
        {
            var request = _httpContextAccessor.HttpContext.Request;
            var baseUrl = $"{request.Scheme}://{request.Host}";

            //var path = Path.Combine(_webRootPath, "images");

            var path = Path.Combine(Environment.CurrentDirectory, "images");

            //string path = "C:/Images/";

            if (!Directory.Exists(path))
            {
                Directory.CreateDirectory(path);
            }

            var orgFileName = file.FileName.Normalize();

            var fileName = DateTime.Now.ToString("yyyyMMddHHmmss_") + orgFileName;

            path = Path.Combine(path, fileName);
            using (var stream = File.Create(path))
            {
                await file.CopyToAsync(stream);
            }

            Stream imageData = file.OpenReadStream();

            var fileSize = file.Length;

            await _userInfoRepository.saveProfileImg(userId, fileSize, orgFileName, fileName, path, imageData);
            
            string imageRelativePath = Path.Combine("images", fileName); // Relative path to the saved image
            string imageUrl = $"{baseUrl}/{imageRelativePath}";

            return imageUrl;
        }
        catch (Exception e)
        {
            _logger.LogError("saveProfile e = " + e);
            throw new ApiException((int)HttpStatusCode.NotFound, HttpStatusCode.NotFound,
                GetEnumDesc.GetEnumDescription(ExceptionEnum.NOT_FOUND_USERID));
        }
    }

public async Task<object> saveProfileImg(string userId, long fileSize, string orgFileName, string fileName,
        string path,
        Stream imageStream)
    {
        using (IDbConnection db = new SqlConnection(_connectionString))
        {
            string USER_ID = userId;
            string ORG_FILE_NAME = orgFileName;
            string FILE_NAME = fileName;
            string PATH = path;
            long FILE_SIZE = fileSize;
            byte[] IMG;

            using (var memoryStream = new MemoryStream())
            {
                await imageStream.CopyToAsync(memoryStream);
                IMG = memoryStream.ToArray();
            }

            string spNm = "dbo.SP_UPLOAD_PROFILE_IMG";

            var parameters = new { USER_ID, ORG_FILE_NAME, FILE_NAME, PATH, FILE_SIZE, IMG };

            var result = await db.ExecuteAsync(spNm, parameters, commandType: CommandType.StoredProcedure);
            
            return result;
        }
    }

CREATE PROCEDURE dbo.SP_UPLOAD_PROFILE_IMG
	@USER_ID varchar(20),
	@ORG_FILE_NAME varchar(100),
	@PATH varchar(100),
	@FILE_NAME varchar(100),
	@FILE_SIZE int, 
	@IMG varbinary(MAX)
AS 
BEGIN

	BEGIN TRAN;

	IF EXISTS (SELECT * FROM USER_PROFILE_IMG 
				WHERE USERID = @USER_ID 
			  ) 
		BEGIN 
		UPDATE USER_PROFILE_IMG 
		   SET ORG_FILE_NAME = @ORG_FILE_NAME, 
		   	   FILE_NAME = @FILE_NAME,
			   FILE_PATH = @PATH, 
			   FILE_SIZE = @FILE_SIZE,
			   PROFILE_IMG = @IMG
		 WHERE USERID = @USER_ID 
		END 
	ELSE 
		BEGIN 
			INSERT INTO USER_PROFILE_IMG (USERID, ORG_FILE_NAME, FILE_NAME, FILE_PATH, FILE_SIZE, PROFILE_IMG)
			VALUES (@USER_ID, @ORG_FILE_NAME, @FILE_NAME, @PATH, @FILE_SIZE, @IMG)
		END
	
	IF @@ERROR <> 0
    BEGIN
        ROLLBACK; — Rollback the transaction if an error occurs
        RETURN;
    END

    COMMIT; — Commit the TRANSACTION
END
```

# login_session_real_time_query
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
