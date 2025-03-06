# swift-file-handling-vanilla

```swift
import Foundation
import UniformTypeIdentifiers

protocol FileServiceProtocol {
    func getFile(key: String) async throws -> String
    func putFile(key: String, fileURL: URL) async throws -> String
    func deleteFile(key: String) async throws -> String
}

final class FileService: FileServiceProtocol {
    
    private var startURL: String {
        Constants.baseURL + EndPoint.file.rawValue
    }
    
    func getFile(key: String) async throws -> String {
        guard let url = URL(string: self.startURL + key) else {
            throw URLError(.badURL)
        }
        let (data, response) = try await URLSession.shared.data(from: url)
        self.prettyPrint(data: data)
        try self.handleResponse(response: response)
        if let jsonResponse = try? JSONSerialization.jsonObject(with: data, options: []) as? [String: Any],
           let urlString = jsonResponse["url"] as? String {
            return urlString
        }
        throw NSError(domain: "Parsing Error", code: 600, userInfo: [NSLocalizedDescriptionKey: "Parsing Error. Field 'url' does not exist or JSONSerialization Failed"])
    }

    func putFile(key: String, fileURL: URL) async throws -> String {
        guard let url = URL(string: self.startURL) else {
            throw URLError(.badURL)
        }
        guard let fileData = try? Data(contentsOf: fileURL) else {
            throw NSError(domain: "Could not read file data", code: 0)
        }
        var request = URLRequest(url: url)
        request.httpMethod = "POST"
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
    
        let body = NSMutableData()
        let uuid: String = UUID().uuidString //for boundary marker
        let boundary: String = "Boundary-\(uuid)"
        request.setValue("multipart/form-data; boundary=\(boundary)", forHTTPHeaderField: "Content-Type")
        let boundaryPrefix = "--\(boundary)\r\n"
        let parameters = ["key": key]
        let fileName = key
        let fieldName = "file"
        let mimeType = self.getMimeType(for: fileURL) ?? ""
        for (key, value) in parameters {
            body.appendString(boundaryPrefix)
            body.appendString("Content-Disposition: form-data; name=\"\(key)\"\r\n\r\n")
            body.appendString("\(value)\r\n")
        }
        body.appendString(boundaryPrefix)
        body.appendString("Content-Disposition: form-data; name=\"\(fieldName)\"; filename=\"\(fileName)\"\r\n")
        body.appendString("Content-Type: \(mimeType)\r\n\r\n")
        body.append(fileData)
        body.appendString("\r\n")
        body.appendString("--\(boundary)--")
        request.httpBody = body as Data
        
        let (data, response) = try await URLSession.shared.data(for: request)
        self.prettyPrint(data: data)
        try self.handleResponse(response: response)
        if let jsonResponse = try? JSONSerialization.jsonObject(with: data, options: []) as? [String: Any],
           let id = jsonResponse["key"] as? String {
            return id
        }
        throw NSError(domain: "InvalidResponse", code: 0, userInfo: [NSLocalizedDescriptionKey: "Unable to parse 'id' from response"])
    }
    
    func deleteFile(key: String) async throws -> String {
        guard let url = URL(string: self.startURL) else {
            throw URLError(.badURL)
        }
        let parameters = ["key": key]
        var request = URLRequest(url: url)
        request.httpMethod = "DELETE"
        request.addValue("application/json", forHTTPHeaderField: "Content-Type")
        request.httpBody = try JSONSerialization.data(withJSONObject: parameters, options: [])
        let (data, response) = try await URLSession.shared.data(for: request)
        self.prettyPrint(data: data)
        try self.handleResponse(response: response)
        // Successfully received valid response, decode the data
        if let jsonResponse = try? JSONSerialization.jsonObject(with: data, options: []) as? [String: Any],
           let key = jsonResponse["key"] as? String {
            return key
        }
        throw NSError(domain: "InvalidResponse", code: 0, userInfo: [NSLocalizedDescriptionKey: "Unable to parse 'id' from response"])
    }
    
    private func handleResponse(response: URLResponse) throws {
        guard let httpResponse = response as? HTTPURLResponse else {
            throw URLError(.badServerResponse)
        }
        switch httpResponse.statusCode {
        case 200..<300:
            break
        case 300..<400:
            throw NSError(domain: "\(FileService.self)-Redirection", code: httpResponse.statusCode, userInfo: [NSLocalizedDescriptionKey: "\(FileService.self)-Redirection error (3xx)"])
        case 400..<500:
            throw NSError(domain: "\(FileService.self)-ClientError", code: httpResponse.statusCode, userInfo: [NSLocalizedDescriptionKey: "\(FileService.self)-Client error (4xx)"])
        case 500..<600:
            throw NSError(domain: "\(FileService.self)-ServerError", code: httpResponse.statusCode, userInfo: [NSLocalizedDescriptionKey: "\(FileService.self)-Server error (5xx)"])
        default:
            throw NSError(domain: "\(FileService.self)-UnknownError", code: httpResponse.statusCode, userInfo: [NSLocalizedDescriptionKey: "\(FileService.self)-Unexpected error"])
        }
    }
    
    private func getMimeType(for fileURL: URL) -> String? {
        let fileExtension = fileURL.pathExtension.lowercased()
        
        // iOS 14 이상에서는 UTType 사용
        if let utType = UTType(filenameExtension: fileExtension) {
            return utType.preferredMIMEType
        }
        
        // 예: iOS 13 이하에서는 기존 방식 사용
        let uti = UTTypeCreatePreferredIdentifierForTag(kUTTagClassFilenameExtension, fileExtension as CFString, nil)
        
        if let utiString = uti?.takeRetainedValue() {
            return UTTypeCopyPreferredTagWithClass(utiString, kUTTagClassMIMEType)?.takeRetainedValue() as String?
        }
        
        return nil
    }

    private func prettyPrint(data: Data) {
#if DEBUG
        do {
            // JSON 디코딩
            if let jsonObject = try JSONSerialization.jsonObject(with: data) as? [String: Any] {
                // Pretty Print
                let prettyData = try JSONSerialization.data(withJSONObject: jsonObject, options: .prettyPrinted)
                if let prettyString = String(data: prettyData, encoding: .utf8) {
                    print(prettyString)
                }
            }
        } catch {
            print(error.localizedDescription)
        }
        
#endif
    }
    
}

fileprivate extension NSMutableData {
    
    
    func appendString(_ string: String) {
        if let data = string.data(using: .utf8) {
            self.append(data)
        }
    }
    
}




```
