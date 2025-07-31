# Lab: Tích hợp Keycloak và midPoint với Keycloak Connector

## Mục tiêu Lab
- Thiết lập môi trường Keycloak và midPoint bằng Docker
- Cài đặt Keycloak Connector để midPoint có thể provisioning user sang Keycloak
- Thực hiện các thao tác CRUD với user và xác minh đồng bộ giữa 2 hệ thống

## Yêu cầu môi trường
- Docker và Docker Compose đã được cài đặt
- Ports 8080, 8081 available trên máy host

## Bước 1: Tạo Docker Network

Tạo network để midPoint và Keycloak có thể giao tiếp với nhau:

```bash
docker network create midpoint_keycloak
```

## Bước 2: Khởi động Keycloak

Chạy container Keycloak với admin account:

> --rm sẽ xoá luôn container khi tắt máy

```bash
sudo docker run --rm --name keycloak -it -p 8080:8080 --net midpoint_keycloak -e KEYCLOAK_ADMIN=admin -e KEYCLOAK_ADMIN_PASSWORD=admin quay.io/keycloak/keycloak:24.0.1 start-dev
```

**Xác minh:** Truy cập http://localhost:8080/admin và đăng nhập với admin/admin

## Bước 3: Tạo Realm trong Keycloak

1. Sau khi đăng nhập vào Keycloak Admin Console
2. Click dropdown ở góc trái (hiện "Master")
3. Click "Add realm"
4. Nhập Name: `test`
5. Click "Create"

**Xác minh:** Realm "test" xuất hiện trong dropdown list

## Bước 4: Chuẩn bị midPoint với Keycloak Connector

Tạo thư mục và download connector:
> Lưu ý: Cần check phiên bản connector phù hợp với Keycloak và Midpoint cài đặt. Check tại category: https://repo1.maven.org/maven2/jp/openstandia/connector/connector-keycloak/

```bash
mkdir -p midpoint-kc/var/icf-connectors
cd midpoint-kc/var/icf-connectors
curl -LO "https://repo1.maven.org/maven2/jp/openstandia/connector/connector-keycloak/1.1.6/connector-keycloak-1.1.6.jar"
cd ../..
```

## Bước 5: Khởi động midPoint

Chạy midPoint container với connector đã download:

> --rm sẽ xoá luôn container khi tắt máy

```bash
docker run --rm --name midpoint -it -p 8081:8080 \
  -v $(pwd)/var:/opt/midpoint/var \
  --net midpoint_keycloak \
  evolveum/midpoint:4.8-alpine
```

**Xác minh:** Truy cập http://localhost:8081 và đăng nhập với Administrator/5ecr3t

## Bước 6: Kiểm tra Connector đã được install

1. Trong midPoint admin console, vào "Repository objects"
2. Trong dropdown, chọn "Connectors"
3. Tìm "KeycloakConnector" trong danh sách

**Xác minh:** KeycloakConnector xuất hiện trong danh sách connectors

## Bước 7: Import Resource Definition

1. Trong midPoint, click menu "Import object"
2. Chọn "Embedded editor" cho "Object import source"
3. Copy và paste XML configuration sau:

```xml
<resource
  xmlns="http://midpoint.evolveum.com/xml/ns/public/common/common-3"
  xmlns:c="http://midpoint.evolveum.com/xml/ns/public/common/common-3"
  xmlns:icfs="http://midpoint.evolveum.com/xml/ns/public/connector/icf-1/resource-schema-3"
  xmlns:org="http://midpoint.evolveum.com/xml/ns/public/common/org-3"
  xmlns:q="http://prism.evolveum.com/xml/ns/public/query-3"
  xmlns:ri="http://midpoint.evolveum.com/xml/ns/public/resource/instance-3"
  xmlns:t="http://prism.evolveum.com/xml/ns/public/types-3">
    <name>keycloak</name>
    <connectorRef relation="org:default" type="c:ConnectorType">
        <filter>
            <q:equal>
                <q:path>c:connectorType</q:path>
                <q:value>jp.openstandia.connector.keycloak.KeycloakConnector</q:value>
            </q:equal>
        </filter>
    </connectorRef>
    <connectorConfiguration xmlns:icfc="http://midpoint.evolveum.com/xml/ns/public/connector/icf-1/connector-schema-3">
        <icfc:resultsHandlerConfiguration>
            <icfc:enableNormalizingResultsHandler>false</icfc:enableNormalizingResultsHandler>
            <icfc:enableFilteredResultsHandler>false</icfc:enableFilteredResultsHandler>
            <icfc:enableAttributesToGetSearchResultsHandler>false</icfc:enableAttributesToGetSearchResultsHandler>
        </icfc:resultsHandlerConfiguration>
        <icfc:configurationProperties xmlns:gen255="http://midpoint.evolveum.com/xml/ns/public/connector/icf-1/bundle/jp.openstandia.connector.connector-keycloak/jp.openstandia.connector.keycloak.KeycloakConnector">
            <gen255:serverUrl>http://keycloak:8080</gen255:serverUrl>
            <gen255:username>admin</gen255:username>
            <gen255:password>admin</gen255:password>
            <gen255:clientId>admin-cli</gen255:clientId>
            <gen255:realmName>master</gen255:realmName>
            <gen255:targetRealmName>test</gen255:targetRealmName>
        </icfc:configurationProperties>
    </connectorConfiguration>
    <schemaHandling>
        <objectType>
            <kind>account</kind>
            <intent>default</intent>
            <displayName>User</displayName>
            <default>true</default>
            <objectClass>ri:user</objectClass>
            <attribute>
                <c:ref>ri:username</c:ref>
                <displayName>Username</displayName>
                <outbound>
                    <strength>strong</strength>
                    <source>
                        <c:path>$focus/name</c:path>
                    </source>
                </outbound>
            </attribute>
            <attribute>
                <c:ref>ri:email</c:ref>
                <displayName>Email</displayName>
                <matchingRule xmlns:mr="http://prism.evolveum.com/xml/ns/public/matching-rule-3">mr:stringIgnoreCase</matchingRule>
                <tolerant>false</tolerant>
                <outbound>
                    <strength>strong</strength>
                    <source>
                        <c:path>$focus/emailAddress</c:path>
                    </source>
                </outbound>
            </attribute>
            <attribute>
                <c:ref>ri:lastName</c:ref>
                <tolerant>false</tolerant>
                <outbound>
                    <strength>strong</strength>
                    <source>
                        <c:path>$focus/familyName</c:path>
                    </source>
                </outbound>
            </attribute>
            <attribute>
                <c:ref>ri:firstName</c:ref>
                <tolerant>false</tolerant>
                <outbound>
                    <strength>strong</strength>
                    <source>
                        <c:path>$focus/givenName</c:path>
                    </source>
                </outbound>
            </attribute>
            <activation>
                <administrativeStatus>
                    <outbound>
                        <strength>strong</strength>
                        <expression>
                            <asIs/>
                        </expression>
                    </outbound>
                </administrativeStatus>
            </activation>
            <credentials>
                <password>
                    <outbound>
                        <strength>strong</strength>
                        <expression>
                           <asIs/>
                        </expression>
                    </outbound>
                </password>
            </credentials>
        </objectType>
    </schemaHandling>
</resource>
```

4. Click "Import object" ở cuối trang

**Xác minh:** Thông báo thành công màu xanh xuất hiện

## Bước 8: Test Connection

1. Vào "Resources" > "List resources"
2. Click vào resource "keycloak"
3. Click "Test connection"

**Xác minh:** Tất cả test results hiển thị màu xanh (SUCCESS)

### ⚠️ Troubleshooting: Nếu gặp lỗi HTTP 404 Not Found

**Nguyên nhân phổ biến:**

1. **Keycloak chưa sẵn sàng hoàn toàn:**
   ```bash
   # Kiểm tra Keycloak logs
   docker logs keycloak
   # Đợi đến khi thấy: "Admin console listening on http://127.0.0.1:9990"
   ```

2. **Network connectivity issues:**
   ```bash
   # Test network connectivity
   docker exec midpoint ping keycloak
   # Should resolve to Keycloak container IP
   ```

3. **Keycloak URL không đúng:**
   - Đảm bảo trong XML config có: `<gen255:serverUrl>http://keycloak:8080</gen255:serverUrl>`
   - Không dùng `localhost` mà phải dùng container name `keycloak`

4. **Realm chưa được tạo:**
   - Đảm bảo realm "test" đã được tạo trong Keycloak
   - Vào http://localhost:8080/admin để kiểm tra

**Các bước khắc phục:**

1. **Restart containers theo thứ tự:**
   ```bash
   # Stop both containers
   docker stop midpoint keycloak
   
   # Start Keycloak first and wait
   docker run --rm --name keycloak -d -p 8080:8080 \
     --net midpoint_keycloak \
     -e KEYCLOAK_USER=admin \
     -e KEYCLOAK_PASSWORD=admin \
     jboss/keycloak:11.0.3
   
   # Wait 2-3 minutes, then check Keycloak is ready
   curl -f http://localhost:8080/realms/master
   
   # Start midPoint
   docker run --rm --name midpoint -d -p 8081:8080 \
     -v $(pwd)/var:/opt/midpoint/var \
     --net midpoint_keycloak \
     evolveum/midpoint:4.2
   ```

2. **Verify Keycloak API accessibility:**
   ```bash
   # Test from midPoint container
   docker exec midpoint curl -f http://keycloak:8080/realms/master
   ```

3. **Re-import resource definition với URL chính xác:**
   - Xóa resource cũ trong midPoint
   - Import lại XML với serverUrl đúng
   - Đảm bảo realm "test" tồn tại

**Validation steps:**
- ✅ Keycloak admin console accessible: http://localhost:8080/auth/admin
- ✅ Realm "test" exists in Keycloak
- ✅ midPoint can ping Keycloak container: `docker exec midpoint ping keycloak`
- ✅ Keycloak API responds: `docker exec midpoint curl http://keycloak:8080/realms/test`

## Bước 9: Tạo User mới trong midPoint

1. Vào "Users" > "New user"
2. Nhập thông tin:
   - **Name**: user1
   - **Given name**: Test
   - **Family name**: User
   - **Email**: user1@example.com
   - **Password > Value**: password

3. Click tab "Assignments"
4. Click assign icon (⊕)
5. Click tab "Resources"
6. Chọn "keycloak" resource
7. Set:
   - **Kind**: Account
   - **Intent**: default
8. Click "Add"

## Bước 10: Preview và Save User

1. Click "Preview changes" để xem những thay đổi sẽ được thực hiện
2. Kiểm tra "Secondary changes" để thấy dữ liệu sẽ được provisioning sang Keycloak
3. Click "Save" để hoàn tất

**Xác minh:** User được tạo thành công trong midPoint

## Bước 11: Kiểm tra User trong Keycloak

1. Vào Keycloak Admin Console
2. Chọn realm "test"
3. Vào "Users" > "View all users"

**Xác minh:** User "user1" xuất hiện trong danh sách users của Keycloak

## Bước 12: Test Login với User mới

1. Truy cập: http://localhost:8080realms/test/account
2. Đăng nhập với user1/password

**Xác minh:** Đăng nhập thành công và hiển thị Account Management page

## Bước 13: Test Update User

1. Trong midPoint, vào "Users" > "All users"
2. Click vào user "user1"
3. Thay đổi "Family name" từ "User" thành "UpdatedUser"
4. Click "Preview changes"
5. Kiểm tra secondary changes
6. Click "Save"

**Xác minh:** 
- Trong Keycloak Admin Console, user1 có Last Name = "UpdatedUser"
- Trong Account Management page (sau khi refresh), Last name đã được cập nhật

## Bước 14: Test Disable User

1. Trong midPoint user detail page
2. Vào "Activation" > "Administrative status"
3. Chọn "Disabled"
4. Click "Preview changes"
5. Kiểm tra secondary changes (ENABLED => DISABLED)
6. Click "Save"

**Xác minh:**
- Trong Keycloak Admin Console, user1 có "User Enabled" = OFF
- Khi thử login vào Account page, sẽ xuất hiện lỗi "Account is disabled"

## Bước 15: Test Delete User

1. Trong midPoint "Users" > "All users"
2. Click action menu (⋮) bên cạnh user1
3. Click "Delete"
4. Confirm "Yes" trong popup

**Xác minh:** 
- User không còn xuất hiện trong midPoint user list
- User không còn xuất hiện trong Keycloak user list

## Kết quả đạt được

Sau khi hoàn thành lab này, bạn đã:

1. ✅ Thiết lập môi trường Keycloak và midPoint
2. ✅ Cài đặt và cấu hình Keycloak Connector
3. ✅ Tạo resource definition cho việc provisioning
4. ✅ Thực hiện provisioning user từ midPoint sang Keycloak
5. ✅ Kiểm tra đồng bộ dữ liệu user (create, update, disable, delete)
6. ✅ Xác minh user có thể đăng nhập vào các ứng dụng được bảo vệ bởi Keycloak

## Mapping thông tin giữa hệ thống

| midPoint Field | Keycloak Field |
|---------------|----------------|
| name          | username       |
| emailAddress  | email          |
| givenName     | firstName      |
| familyName    | lastName       |
| Administrative Status | User Enabled |
| Password      | Password       |

## Cleanup

Để dọn dẹp môi trường sau khi làm lab:

```bash
# Stop containers
docker stop keycloak midpoint

# Remove network
docker network rm midpoint_keycloak

# Remove downloaded files
rm -rf midpoint-kc/
```