+++
title = 'Canvas 2d - Check Point in Rects when Mouse Moving'
date = 2024-06-29T09:37:16+07:00
draft = false
+++

## Giới thiệu

Tuần trước mình đang hoàn thiện tính năng cho trang demo sản phẩm của công ty AI PRO, trong đó có một vấn đề khá hay liên quan đến việc kiểm tra va chạm hình học trong canvas 2d, thiết nghĩ nên chia sẻ ra đây. Đầu tiên là để lưu nhớ lại, sau là mong muốn nhận được góp ý của người đọc để có thể cải thiện sản phẩm hơn nữa.

## Nghiệp vụ

Cụ thể, khách hàng sẽ gửi một danh sách hình ảnh (ví dụ là ảnh chụp hóa đơn) lên máy chủ, máy chủ sẽ xử lý hình ảnh, nhận dạng các đoạn chữ viết, số, ký tự v.v. trong ảnh, rồi trả về kết quả. Trình duyệt sẽ xử lý và hiển thị kết quả như hình sau.

### Ảnh 1 - Kết quả hiển thị cho khách hàng sau khi được hệ thống AI xử lý (_một phần giao diện thực tế_)

![Kết quả hiển thị cho khách hàng sau khi được hệ thống AI xử lý hình ảnh](/images/posts/canvas-2d-check-point-in-rects-when-mouse-moving/image-canvas-2d-result-rendering.png "Kết quả hiển thị cho khách hàng sau khi được hệ thống AI xử lý hình ảnh")

Kết quả trả về từ máy chủ gồm nhiều thông tin, trong đó cái ta cần quan tâm ở đây là một mảng mô tả các đoạn văn bản mà hệ thống AI nhận diện được, như vị trí văn bản trên ảnh và văn bản là gì.

#### Định dạng dữ liệu trả về cho hình ảnh trên

```json
{
  "data": {
    // ... other properties
    "result": [
      {
        // ... other properties
        "h": 14,
        "id": 0,
        "text": "インボイスを作成した",
        "w": 89,
        "x": 576,
        "y": 8
      },
      {
        // ... other properties
        "h": 11,
        "id": 1,
        "text": "年月日を記入",
        "w": 61,
        "x": 574,
        "y": 22
      }
      // ... other items
    ]
  }
}
```

> Mỗi item thể hiện một đoạn văn bản tương ứng trên ảnh, là một hình chữ nhật có vị trí trên trái (x, y) và chiều rộng và cao (w, h) cùng với đoạn văn bản (text) mà hình chữ nhật đó chứa. Đây là vị trí tương đối so với ảnh gốc, ở dưới khi xác định va chạm với con trỏ chuột là vị trí tương đối với màn hình, chúng ta cần phải làm một bước chuyển đổi từ hệ tọa độ màn hình sang hệ tọa độ của ảnh.

Trình duyệt dựa vào dữ liệu trên để vẽ ra các ô chữ nhật màu đỏ, đánh dấu vị trí cho các văn bản trên hình ảnh, bằng cách dùng một thẻ canvas chèn trên ảnh. Trong thẻ canvas sẽ gọi đối tượng context 2d để vẽ các hình chữ nhật.

```tsx {lineNos=inline tabWidth=2}
<div
  style={{
    position: "relative",
    overflow: "auto",
    height: windowHeight,
    margin: "10px",
  }}
>
  <img
    ref={imageRef}
    src={imageUrls}
    style={{
      width: "100%",
      position: "absolute",
      zIndex: 1,
    }}
    onLoad={handleImageLoad}
  />
  <canvas
    ref={canvasRef}
    style={{
      width: "100%",
      position: "absolute",
      zIndex: 2,
    }}
    onClick={handleCanvasClick}
    onMouseMove={handleCanvasMouseMove}
    onMouseLeave={handleCanvasMouseLeave}
  ></canvas>
</div>
```

Ta duyệt qua danh sách các phần tử là đối tượng thể hiện vị trí của text do server trả về, rồi dùng canvas 2d để vẽ các cạnh và tô màu đỏ.

```tsx {lineNos=inline tabWidth=2}
const canvas = canvasRef.current;
if (!canvas) return;
const ctx = canvas.getContext("2d");
if (!ctx) return;
ctx.clearRect(0, 0, canvas.width, canvas.height);
boxes.forEach((box: any) => drawTextBox(ctx, box));
```

```tsx {lineNos=inline tabWidth=2}
const drawTextBox = (ctx: CanvasRenderingContext2D, box: any) => {
  if (!ctx || !box) return;
  ctx.beginPath();
  ctx.rect(box.x, box.y, box.w, box.h);
  ctx.strokeStyle = "red";
  ctx.lineWidth = 1;
  ctx.stroke();
};
```

Tóm lại, đến đây ta đã vẽ được các hình chữ nhật thể hiện vị trí của các đoạn text trên ảnh, việc cần làm tiếp theo là làm sao để khách hàng có thể chọn được các đoạn văn bản đó bằng cách di chuột vào ô chữ nhật màu đỏ và nhấn, các đoạn text tương ứng với ô đó sẽ hiển thị vào ô input đã chọn bên cạnh. Khi di chuột vào trong phạm vi của hình chữ nhật nào thì con trỏ sẽ hiển thị dạng "pointer", khi ra khỏi thì trở về dạng tự động "auto", để báo cho khách hàng nhận biết rằng họ có thể chọn một văn bản hay không.

## Giải pháp

Đầu tiên, trong thẻ canvas ta bắt sự kiện di chuột `onMouseMove`, trong hàm xử lý sự kiện di chuột `handleCanvasMouseMove` ta lấy vị trí của con trỏ, duyệt qua danh sách các hình chữ nhật, kiểm tra xem con trỏ có ở trong hình chữ nhật nào không, nếu có trả về hình chữ nhật đó, đồng thời chuyển `style.cursor` của thẻ canvas thành `pointer`.

Tiếp theo, nếu khách hàng nhấn vào đoạn văn bản đó, dựa vào hình chữ nhật ta nhận được ở bước trên, lấy ra `text` và hiển thị vào ô input.

Chi tiết các bước:

- B1. Bắt sự kiện `onMouseMove` của thẻ canvas, qua đó lấy được vị trí hiện tại (x,y) của con trỏ đồng thời chuyển đổi vị trí của con trỏ từ tọa độ màn hình sang tọa độ trong canvas context 2d.

  ```tsx {lineNos=inline tabWidth=2}
  const getMousePosition = (e: MouseEvent): IPoint | null => {
    const { clientX, clientY } = e;
    const canvas = canvasRef.current;
    const resizeRate = getResizeRate();
    if (!canvas || !resizeRate) return null;
    const canvasRect = canvas.getBoundingClientRect();
    const x = (clientX - canvasRect.x) / resizeRate.x;
    const y = (clientY - canvasRect.y) / resizeRate.y;
    return { x, y };
  };

  const getResizeRate = () => {
    const canvas = canvasRef.current;
    const image = imageRef.current;
    if (
      !canvas ||
      !image ||
      image.naturalWidth < 0 ||
      image.naturalHeight < 0
    ) {
      return;
    }
    return {
      x: canvas.clientWidth / image.naturalWidth,
      y: canvas.clientHeight / image.naturalHeight,
    };
  };
  ```

  > - Dòng 2: Ở sự kiện `MouseEvent` ta lấy ra được hai thông số `clientX` và `clientY`. Đây là vị trí của con trỏ chuột tính theo hệ tọa độ màn hình. Ta cần phải chuyển hai thông số này sang hệ tọa độ trong ảnh gốc.
  > - Dòng 4: Ta lấy ra resizeRate = {x, y} là đối tượng chứa tỉ lệ kích thước của ảnh hiển thị (cũng là kích thước của thẻ canvas) so với ảnh gốc theo chiều x (ngang) và chiều y (dài). Xem hàm getResizeRate() để rõ hơn cách tính. Vì trang web hiển thị trên nhiều màn hình có kích thước khác nhau, nên ảnh gốc cần được thay đổi kích thước hiển thị sao cho phù hợp nhất trên màn hình đó.
  > - Dòng 6: Ta lấy ra canvasRect = {x, y, ...} là đối tượng chứa vị trí của thẻ canvas (cũng là vị trí của ảnh hiển thị) tính theo hệ tọa độ màn hình.
  > - Dòng 7,8: Đến đây, ta tính ra được vị trí của con trỏ chuột theo hệ tọa độ ảnh gốc theo mỗi chiều bằng cách lấy vị trí của tọa độ màn hình trừ đi vị trí của ảnh hiển thị, rồi chia cho tỉ lệ thay đổi kích thước của ảnh hiển thị.  
  >   `x = (clientX - canvasRect.x) / resizeRate.x;`  
  >   `y = (clientY - canvasRect.y) / resizeRate.y;`  
  > - Xem ảnh sau để rõ hơn:
  > ![Chuyển đổi vị trí con trỏ chuột từ hệ tọa độ màn hình sang hệ tọa độ ảnh gốc](/images/posts/canvas-2d-check-point-in-rects-when-mouse-moving/transfer-mouse-position-from-screen-to-origin-image.drawio.png "Chuyển đổi vị trí con trỏ chuột từ hệ tọa độ màn hình sang hệ tọa độ ảnh gốc")

- B2. Duyệt các hình chữ nhật để kiểm tra va chạm. Sau khi ta đưa vị trí con trỏ về cùng hệ tọa độ với vị trí của các hình chữ nhật, ta có thể kiểm tra xem một điểm có thuộc một hình chữ nhật không theo cách sau:

  ```tsx {lineNos=inline tabWidth=2}
  ... // more code here
      for (const rect of rects) {
        if (GeometryUtils.isPointInRect(point, rect)) {
          return rect;
        }
      }
  ... // more code here

  static isPointInRect(point: IPoint, rect: IRect): boolean {
    if (!point || !rect) return false;
    const top = rect.y;
    const left = rect.x;
    const bottom = top + rect.h;
    const right = left + rect.w;
    return !(
      point.x < left ||
      point.x > right ||
      point.y < top ||
      point.y > bottom
    );
  }
  ```

- B3.1 Nếu con trỏ ở trong một hình chữ nhật, chuyển thuộc tính css `styles.cursor` của thẻ canvas thành `pointer` đồng thời lưu lại hình chữ nhật đó.
- B3.2 Nếu con trỏ không ở trong một hình chữ nhật nào, chuyển thuộc tính css `styles.cursor` của thẻ canvas thành `auto` đồng thời bỏ lưu lại hình chữ nhật đã va chạm trước đó nếu có.
- B4. Nếu khách hàng nhấn vào văn bản khi con trỏ đang ở trong một hình chữ nhật, trong sự kiện `onClick` ta lấy văn bản của hình chữ nhật đã lưu trước đó (thuộc tính `text`) rồi hiển thị ra ô input.

Đến đây, về cơ bản vấn đề đã được giải quyết. Tuy nhiên, ở bước B2 ta nhận thấy rằng cứ mỗi khi phát sinh sự kiện `onMouseMove` thì ta lại phải duyệt hết danh sách các hình chữ nhật để kiểm tra va chạm.

Nếu số lượng các hình chữ nhật ít và máy tính cấu hình cao thì việc này không đáng kể. Tuy nhiên giả sử số lượng hình chữ nhật rất nhiều, và ta cần tối ưu để đỡ tiêu tốn tài nguyên tính toán của máy tính, thì ta có thể làm tốt hơn không?

## Tối ưu

Cách 1: Ta có thể giảm số lần tính toán bằng cách thêm khoảng thời gian trễ khi nhận sự kiện `onMouseMove`, tức là sau khoảng thời gian nhất định (ví dụ 16ms) thì ta mới tính toán lại, còn chưa hết 16ms thì ta bỏ qua không tính toán. Việc này có thể phát sinh vấn đề về độ mượt mà của trang web, có thể gây cảm giác lag cho người dùng.

Cách 2: Ta giảm số lượng hình chữ nhật phải duyệt trong mỗi lần tính toán. Giả sử bằng một cách nào đó ta biết rằng chỉ một số lượng nhỏ m hình chữ nhật trong tổng số n hình chữ nhật đã cho là có khả năng va chạm với con trỏ ở thời điểm hiện tại, ta chỉ cần duyệt m hình chữ nhật thôi (m nhỏ hơn n rất nhiều).

Sau đây ta sẽ đi theo cách thứ 2.

![Con trỏ chuột và các hình chữ nhật trong ảnh](/images/posts/canvas-2d-check-point-in-rects-when-mouse-moving/mouse-and-rects-in-image.drawio.png "Con trỏ chuột và các hình chữ nhật trong ảnh")

Thật vậy, nhìn ảnh ta dễ thấy rằng ở mỗi sự kiện `onMouseMove` con trỏ chuột chỉ có khả năng va chạm với một nhóm nhỏ gồm m hình chữ nhật lân cận nó (được khoanh vùng bằng ô vuông nét đứt). Như vậy ta chỉ cần kiểm tra va chạm của con trỏ với m hình chữ nhật này (m ở đây là 6) so với tổng số n hình chữ nhật (n ở đây là 20).

Điều này dẫn ta đến một ý tưởng, ta sẽ chia ảnh thành mạng lưới gồm u x v ô chữ nhật. Sau đó với mỗi ô trong mạng lưới, ta sẽ xếp các hình chữ nhật mà ô đó chứa một phần hoặc toàn bộ hình chữ nhật đó vào bên trong, ta gọi số hình chữ nhật này là m. Sau đó, ở mỗi sự kiện `onMouseMove`, đầu tiên ta sẽ kiểm tra xem con trỏ chuột đang ở trong ô nào, tiếp theo ta chỉ cần duyệt m hình chữ nhật trong ô đó để kiểm tra va chạm.

![Con trỏ chuột và các hình chữ nhật trong mạng lưới chia ô của ảnh](/images/posts/canvas-2d-check-point-in-rects-when-mouse-moving/mouse-and-rects-in-grid-system-image.drawio.png "Con trỏ chuột và các hình chữ nhật trong mạng lưới chia ô của ảnh")

Bây giờ ta sẽ xếp các hình chữ nhật vào từng ô, ta đánh số mỗi ô là một cặp (u,v):  

```txt
(1,1)=[1]     (1,2)=[1,2,4]       (1,3)=[2,4,5]            (1,4)=[3,5,6]
(2,1)=[7,12]  (2,2)=[7,8,10,12]   (2,3)=[8,9,10,11,13,14]  (2,4)=[9,11,14]
(3,1)=[15]    (3,2)=[15,17]       (3,3)=[13,14,16,17]      (3,4)=[14,16,17]
(4,1)=[]      (4,2)=[17]          (4,3)=[17,18,19,20]      (4,4)=[17,19,20]
```

> Có một điểm lưu ý nhỏ là ta vẫn xét một hình chữ nhật thuộc một ô khi mà chỉ một cạnh của nó trùng khớp với cạnh của ô.  
> u và v có thể khác nhau tùy vào cách ta phân chia.

Theo ảnh, con trỏ chuột đang ở ô (2,3) nên ta chỉ phải kiểm tra va chạm với 6 hình chữ nhật là 8, 9, 10, 11, 13, và 14. Tổng số hình chữ nhật càng nhiều thì việc phân chia này càng có ý nghĩa.

Lý thuyết cơ bản đã xong, tiếp theo ta sẽ phân tích chi tiết cách làm.

## Phân tích

Ta phải làm 3 việc sau,

1. Tìm ra một cách phân chia ảnh thành lưới các ô sao cho tối ưu nhất.

2. Xếp các hình chữ nhật vào từng ô và lưu trữ trong một cấu trúc dữ liệu sao cho việc truy xuất đạt tốc độ nhanh nhất và ít tốn bộ nhớ nhất.

3. Tìm ô mà con trỏ chuột đang nằm trong mỗi sự kiện `onMouseMove`. Qua đó tìm ra danh sách m hình chữ nhật nằm trong ô đó.

Khi đã xong 3 việc trên, thì tiếp theo ta chỉ phải duyệt một danh sách m rất nhỏ hình chữ nhật để kiểm tra va chạm.

### 1. Tìm ra một cách phân chia ảnh thành lưới các ô sao cho tối ưu nhất

Tối ưu nhất ở đây có thể hiểu là:

1.1. Số lượng hình chữ nhật (m) trong mỗi ô phải đủ nhỏ so với tổng số hình chữ nhật (n). Nếu không thì việc phân chia này không có ý nghĩa lắm.  
  Điều này khiến ta suy nghĩ rằng ta càng chia kích thước của các ô càng nhỏ càng tốt, vì khi đó m sẽ càng nhỏ. Nhưng liệu rằng điều đó có thực sự tốt?

![Con trỏ chuột và các hình chữ nhật trong nhiều mạng lưới chia ô của ảnh](/images/posts/canvas-2d-check-point-in-rects-when-mouse-moving/mouse-and-rects-in-image-grid-systems.drawio.png "Con trỏ chuột và các hình chữ nhật trong nhiều mạng lưới chia ô của ảnh")

Quan sát ảnh 1 ta thấy khi không phân chia lưới ô, nghĩa là ta coi cả ảnh là một ô chứa toàn bộ hình chữ nhật, đây cũng là giới hạn trên của việc phân chia.

Ở ảnh 2, ta chia thành 4x4=16 ô vuông, trung bình có 43:16=2.68 hình chữ nhật trong một ô, tỷ lệ vào khoảng 2.68:20=13.4% so với tổng số 20 hình chữ nhật. Một con số giảm đáng kể.

Tiếp theo ảnh 3, ta chia nhỏ hơn nữa thành 8x8=64 ô vuông, tỷ lệ càng giảm hơn nữa, 77:64=1.2 hình chữ nhật trong một ô, tỷ lệ vào khoảng 1.2:20=6% so với tổng số 20 hình chữ nhật. Đến đây mọi chuyện vẫn tốt đẹp.

Nhưng ở ảnh 4, khi ta chia thành 16x16=256 ô vuông, tỷ lệ không giảm là bao mà lại xuất hiện rất nhiều ô trống (ô không có hình chữ nhật nào). Đồng thời ta để ý thấy là các hình chữ nhật bị phân mảnh trong rất nhiều ô vuông, điều này dẫn đến sự trùng lặp hình chữ nhật ở các ô.

Kết luận, có một giới hạn dưới cho việc phân chia lưới các ô. Điều này dẫn ta đến ý thứ 2 của việc tối ưu.

1.2. Tổng số ô trong lưới không quá lớn cũng như các hình chữ nhật không bị trùng lặp giữa các ô. Điều này để chuẩn bị cho việc thứ 2 trong số 3 việc ta phải làm, tức là ta sẽ tốn ít bộ nhớ để lưu trữ các ô và danh sách các hình chữ nhật trong ô.

![Con trỏ chuột và các hình chữ nhật trong mạng lưới chia ô tối ưu của ảnh](/images/posts/canvas-2d-check-point-in-rects-when-mouse-moving/mouse-and-rects-in-image-grid-system-optimize.drawio.png "Con trỏ chuột và các hình chữ nhật trong mạng lưới chia ô tối ưu của ảnh")

Giả sử một ảnh tối ưu đơn giản, ta chia thành 10x3=30 ô, trung bình có 55:30=1.8 hình chữ nhật trong một ô, tỷ lệ vào khoảng 1.8:20=9% so với tổng số 20 hình chữ nhật. Có một sự cân bằng giữa ảnh 5 so với ảnh 2 và ảnh 3.

Nói cách khác là ta đang cân bằng giữa tốc độ và bộ nhớ của thuật toán. Để làm điều này, tức là tìm ra giới hạn dưới cho của việc phân chia ô, ta có một nhận xét rằng một cách phân chia lý tưởng là các ô nên bao trọn các hình chữ nhật mà nó chứa. Nói một cách khác thì các hình chữ nhật không nên bị phân mảnh quá nhiều giữa các ô.

Bạn đọc vẫn theo kịp chứ? Dựa vào nhận xét bên trên, suy ra độ rộng của một ô cần lớn hơn độ rộng của đa số các hình chữ nhật, và chiều cao của một ô cũng cần lớn hơn chiều cao của đa số các hình chữ nhật. Lớn hơn ở đây cần hiểu là lớn hơn ở mức tối thiểu và đa số ở đây có thể hiểu là 50% hoặc 75% (tùy chọn) trong tổng số các hình chữ nhật. Sau đây, để cho dễ thì ta sẽ chọn con số 50%, tức là tìm ra độ rộng trung bình và chiều cao trung bình của các hình chữ nhật. (*)

Ví dụ ở ảnh sau:

![Con trỏ chuột và các hình chữ nhật trong mạng lưới chia ô tối ưu của ảnh với background](/images/posts/canvas-2d-check-point-in-rects-when-mouse-moving/mouse-and-rects-in-image-grid-system-with-background.drawio.png "Con trỏ chuột và các hình chữ nhật trong mạng lưới chia ô tối ưu của ảnh với background")

Để minh họa với ảnh trên, ước lượng một cách đơn giản ta có:

```txt
w=4 (có 7 hình chữ nhật)
w=8 (có 7 hình chữ nhật)
w=10 (có 1 hình chữ nhật)
w=14 (có 4 hình chữ nhật)
w=18 (có 1 hình chữ nhật)
=> averageW = (4*7 + 8*7 + 10*1 + 14*4 + 18*1) / 20 = 8.4

h=2 (có 20 hình chữ nhật)
=> averageH = 2
```

Vậy chiều rộng của ô cần lớn hơn 8.4 và chiều cao của ô cần lớn hơn 2. Giả sử ta chọn chiều rộng ô là 16 (cellW) và chiều cao là 4 (cellH) (ở phần sau sẽ giải thích tại sao ta lại chọn như vậy ). Ta vẫn có 10x3=30 ô, tuy nhiên lần này chiều rộng không phải chia đều như ảnh trước (đây chính xác là cách thuật toán hoạt động).

Tiếp theo ta cần phải đưa các hình chữ nhật vào các ô.

### 2. Xếp các hình chữ nhật vào từng ô và lưu trữ trong một cấu trúc dữ liệu sao cho việc truy xuất đạt tốc độ nhanh nhất và ít tốn bộ nhớ nhất

#### Xếp các hình chữ nhật vào từng ô

Ta sẽ duyệt qua tất cả các hình chữ nhật, và tính toán xem nó thuộc vào các ô nào (**). Ví dụ với hình chữ nhật thứ 15, hình này có:  
`{x, y, w, h} = {5, 23, 14, 2}`

Ta kiểm tra 4 điểm góc của hình chữ nhật này, rồi tính xem các điểm đó thuộc về ô nào thì ta thêm hình chữ nhật này vào danh sách hình chữ nhật của ô đó và các ô nằm giữa các ô đó. Ví dụ:

điểm trên trái: {x, y} = {5, 23} thuộc vào ô (v, u) = {0, 5} với:  
`v = x / cellW = 5 / 16 = 0`  
`u = y / cellH = 23 / 4 = 5`  
(phép chia lấy phần nguyên)

tương tự với điểm trên phải: {x, y} = {19, 23} thuộc vào ô (v, u) = {1, 5}  
điểm dưới trái: {x, y} = {5, 25} thuộc vào ô (v, u) = {0, 6}  
điểm dưới phải: {x, y} = {19, 25} thuộc vào ô (v, u) = {1, 6}  

Tóm lại tất cả các ô từ {0,5}, {0,6}, {1,5}, {1,6} đều chứa hình chữ nhật thứ 15, ta đều phải thêm hình chữ nhật 15 vào danh sách hình chữ nhật của các ô đó. Cụ thể,  

```js
for (let lowX = 0; lowX <= 1; lowX++) {
  for (let lowY = 5; lowY <= 6; lowY++) {
    // Cập nhật danh sách hình chữ nhật của ô (lowX, lowY)
  }
}
```

![Con trỏ chuột và các hình chữ nhật trong mạng lưới chia ô tối ưu của ảnh với hình chữ nhật 15 kích thước lớn](/images/posts/canvas-2d-check-point-in-rects-when-mouse-moving/mouse-and-rects-in-image-rect-15-bigsize.drawio.png "Con trỏ chuột và các hình chữ nhật trong mạng lưới chia ô tối ưu của ảnh với hình chữ nhật 15 kích thước lớn")

Ảnh trên với hình chữ nhật 15 vẽ to bao trùm nhiều ô trong lưới, ta sẽ hiểu vì sao ta phải tính các ô chứa 4 góc rồi duyệt từng ô để cập nhật lại danh sách hình chữ nhật trong từng ô đó.

#### Lưu trữ trong một cấu trúc dữ liệu sao cho việc truy xuất đạt tốc độ nhanh nhất và ít tốn bộ nhớ nhất

Hình ảnh phân chia lưới làm ta dễ dàng liên tưởng đến một cấu trúc dữ liệu dạng ma trận, trong đó mỗi phần tử của ma trận đại diện cho một ô là một mảng chứa các hình chữ nhật của ô đó.

- Việc truy xuất đến từng ô thông qua chỉ số của ô có độ phức tạp O(1).
- Thao tác cập nhật danh sách hình chữ nhật của ô chỉ là thao tác thêm vào cuối danh sách (push) nên cũng có độ phức tạp là O(1).
- Kích thước ma trận hay độ phức tạp không gian là O(u.v.m) đã được ta tối ưu ở việc 1. Đây cũng là một phần lý do vì sao ta cần phải cân bằng giữa việc chia nhỏ lưới (u.v) và số lượng trung bình hình chữ nhật trong một ô (m).

### 3. Tìm ô mà con trỏ chuột đang nằm trong mỗi sự kiện `onMouseMove`. Qua đó tìm ra danh sách m hình chữ nhật nằm trong ô đó

#### Tìm ô mà con trỏ chuột đang nằm trong mỗi sự kiện `onMouseMove`

Công việc tương tự như khi ta tính toán 4 điểm góc của hình chữ nhật thuộc ô nào ở bên trên. Ở đây, theo ảnh giả sử con trỏ chuột đang ở vị trí `{x, y} = {23, 19}`, vị trí này là vị trí sau khi ta đã chuyển đổi từ tọa độ màn hình sang tọa độ ảnh gốc, suy ra nó sẽ thuộc vào ô `(v, u) = (1, 4)`.

#### Qua đó tìm ra danh sách m hình chữ nhật nằm trong ô đó

Dựa vào ma trận đã tính toán bên trên, ta lấy ra được danh sách các hình chữ nhật của ô (1, 4) là [10, 11, 12, 13, 14]. Đến đây công việc quay về bước B2 với một số lượng hình chữ nhật ít hơn rất nhiều.

> Lưu ý: Ta chỉ cần tính toán ma trận một lần lúc tải trang (***)

Phân tích đã xong, bây giờ đến lúc lập trình. Tuy nhiên, bài viết đến đây cũng khá là dài. Mình sẽ dành phần lập trình ở phần 2.

Cảm ơn bạn đã đọc đến đây.

## Tôi hỏi và bạn trả lời

1. (*) Khi tính toán độ rộng trung bình và chiều cao trung bình của các hình chữ nhật, ta đã chọn con số 50%, nếu ta tăng hoặc giảm con số này giả sử 25% hoặc 75% thì điều gì sẽ thay đổi? Kích thước ma trận (u.v) và số hình chữ nhật trung bình m trong 1 ô có thay đổi không?

2. (**) Khi xếp các hình chữ nhật vào mỗi ô, ta đã chọn cách duyệt từng hình chữ nhật (n), rồi xếp chúng lần lượt vào các ô tương ứng. Nếu ta làm ngược lại, ta duyệt ma trận ô (u.v), rồi sau đó tìm xem hình chữ nhật nào thuộc ô đó, thì độ phức tạp tính toán là bao nhiêu? Cách nào tốt hơn?

3. (***) Tại sao lại nói ta chỉ cần tính toán ma trận một lần lúc tải trang? Nếu kích thước màn hình thay đổi (giả sử ta xoay ngang màn hình) thì có cần phải tính toán lại ma trận không? Tại sao?

4. Ở B1, trong phần Giải pháp, với mỗi sự kiện `onMouseMove` ta đều phải chuyển đổi vị trí con trỏ chuột từ tọa độ màn hình sang tọa độ ảnh gốc. Ta có thể làm ngược lại không?, tức là chuyển hết vị trí các hình chữ nhật từ tọa độ ảnh gốc sang tọa độ màn hình, rồi kiểm tra và chạm luôn với vị trí của con trỏ mà không phải chuyển đổi ở mỗi sự kiện `onMouseMove`. Điều này có liên quan gì đến câu hỏi (***) không?
